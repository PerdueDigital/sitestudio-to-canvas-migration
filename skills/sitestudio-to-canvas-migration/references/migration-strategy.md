# Migration Strategy: Sequencing, Risks, and Decommission

The other references describe **how** to convert each piece. This one covers **when** and **in what order** — plus the supporting work that gets missed if you focus only on components.

## Table of Contents

1. [Migration Phases](#migration-phases)
2. [Discovery & Inventory](#discovery--inventory)
3. [Site Studio Templates (`cohesion_templates.*`)](#site-studio-templates-cohesion_templates)
4. [Text Format / CKEditor Considerations](#text-format--ckeditor-considerations)
5. [Base Theme Change](#base-theme-change)
6. [Site Studio Sync Packages & Style Helpers](#site-studio-sync-packages--style-helpers)
7. [Acquia DAM Integration](#acquia-dam-integration)
8. [Multilingual / Translations](#multilingual--translations)
9. [Testing & Visual Regression](#testing--visual-regression)
10. [Module Uninstall Sequence](#module-uninstall-sequence)
11. [Performance & Caching Differences](#performance--caching-differences)

---

## Migration Phases

A clean phasing keeps the site working at every step. **Do not start uninstalling Site Studio modules until phase 5.**

1. **Foundation** — install Canvas (`drupal/canvas`), confirm assets built (`ui/dist/`), set up your CSS build (e.g. Tailwind v4) in each theme, create your custom SDC module, ship one trivial SDC end-to-end (Heading or Button) and verify it appears in `/admin/appearance/component` and can be placed via `/canvas/editor/...`.
2. **Tokens & base styles** — port Cohesion color/typography/breakpoint tokens to Tailwind `@theme` and `@layer base` (see `@references/style-migration.md`). Canvas and Site Studio render side-by-side without visual collision at this point.
3. **Component port** — convert custom code components first (see `@references/component-migration.md`), then built-in elements (`@references/builtin-elements.md`), then UI-created components (`@references/ui-component-migration.md`). Add Component config entities (auto on cache rebuild) and export them to config.
4. **Content migration** — add a `component_tree` field (e.g. `field_canvas_layout`) to each affected bundle; run the migration to populate it from `layout_canvas` JSON (see `@references/content-migration.md`). **Both fields exist on the bundle simultaneously.**
5. **Cutover** — per bundle, switch the entity view display to render `field_canvas_layout` instead of `layout_canvas`. Validate visually + functionally. Once stable, delete the `layout_canvas` field.
6. **Decommission** — uninstall Site Studio modules in dependency order (see [Module Uninstall Sequence](#module-uninstall-sequence)), then `composer remove acquia/cohesion`.

Phases 3 and 4 can run in parallel once foundation is in place. Phase 5 is per-bundle, not all-at-once. On a multisite codebase, run this phasing per site — a site can be in phase 5 while another is still in phase 2.

---

## Discovery & Inventory

Before estimating, run a discovery pass on each codebase/site. Capture counts to a checked-in `migration-inventory.md` or ticket — this becomes the source for the project-specific overlay skill described in the main SKILL.md.

```bash
# Custom code components.
find docroot/modules/custom -name '*.custom_component.yml' | sort

# UI-created components per site config dir.
ls config/{site}/{site-studio-config-dir}/cohesion_elements.cohesion_component.*.yml | wc -l

# Helpers (need expansion before migration).
ls config/{site}/{site-studio-config-dir}/cohesion_elements.cohesion_helper.*.yml | wc -l

# Templates.
ls config/{site}/{site-studio-config-dir}/cohesion_templates.*.yml | wc -l

# Nodes using Layout Canvas — by bundle.
drush sql:query "SELECT bundle, COUNT(*) c FROM node__layout_canvas WHERE layout_canvas_value IS NOT NULL GROUP BY bundle;"

# Unique built-in element uids actually used across all layout_canvas content.
drush php-eval '
  $rows = \Drupal::database()->query("SELECT layout_canvas_value FROM node__layout_canvas WHERE layout_canvas_value IS NOT NULL")->fetchCol();
  $uids = [];
  foreach ($rows as $row) {
    $d = json_decode($row, TRUE);
    foreach ($d["canvas"] ?? [] as $i) {
      $uids[$i["uid"] ?? "?"] = ($uids[$i["uid"] ?? "?"] ?? 0) + 1;
    }
  }
  arsort($uids); print_r($uids);
'
```

The last command produces an actual-usage histogram — converting the 80%-tail components first cuts the most work. Components used zero times can be skipped entirely.

---

## Site Studio Templates (`cohesion_templates.*`)

`cohesion_templates.cohesion_content_templates.*` and `cohesion_templates.cohesion_master_templates.*` define how Site Studio renders **entity view modes** (e.g. `node_article_full`) — not page content. These are independent of `layout_canvas` field data.

Migration paths:

- **Master template** (the wrapper providing site header/footer/layout): replace with the new base theme's `page.html.twig` plus blocks placed in regions. Canvas does not own page chrome.
- **Content templates** (per-bundle, per-view-mode): in Canvas, render these via a normal entity view display + theme template (`node--{bundle}--{view-mode}.html.twig`). If the content template contained Site Studio components, port the equivalent SDCs into the twig file, or render a `component_tree` field there. Canvas also supports **Content Templates** (a Canvas concept distinct from Site Studio's) — config entity `canvas.content_template.*` — used when you want a Canvas-edited template shared across nodes of a bundle. Reach for this only if authors need to edit the template; otherwise stick with twig.
- **View modes used only for embedded display** (cards, teasers): often easiest to rebuild as twig + your CSS framework's utilities without involving Canvas at all.

Inventory every `cohesion_templates.*` file per site and decide twig-vs-Canvas-Content-Template before phase 5.

---

## Text Format / CKEditor Considerations

Site Studio installs `sitestudio_legacy_ckeditor` and may have content stored in text formats with WYSIWYG markup containing Site Studio-aware shortcodes or classes.

Before disabling Site Studio modules:

- Identify text formats that depend on Site Studio filters: `drush config:list --filter='filter.format'` then `drush config:get filter.format.{id}`. Look for `cohesion_*` filters.
- Replace any `cohesion_*` filter with a core/contrib equivalent (or remove if redundant). Re-save text formats.
- If body content references `coh-style-*` classes that won't exist post-Site-Studio, decide: leave them inert (no styling) or run a one-off sed/regex pass on body fields to map to your CSS framework's utilities. Test on a clone of production data first.
- CKEditor 5 is the default in Drupal 11. If any format still uses CKEditor 4 via `sitestudio_legacy_ckeditor`, switch to CKEditor 5 before uninstalling that submodule.

---

## Base Theme Change

Site Studio sites typically use `cohesion_theme` or a custom theme that inherits from it. Canvas does not require a specific base theme, but components rely on your CSS framework's utility classes being present.

Approach:

- Keep the existing custom theme. Remove `base theme: cohesion_theme` (or any Cohesion-derived base). Switch to `base theme: stable9` or a Drupal core base, or no base theme.
- Move your CSS build output into `dist/` declared in `{theme}.libraries.yml`; ensure it's attached globally via `*.info.yml` `libraries:` or via `hook_page_attachments_alter()` for the active theme.
- The `canvas_stark` theme shipped with Canvas is a minimal admin theme — useful as a reference, not a public-facing theme.
- Verify region definitions: the new theme must declare the regions blocks are placed into (`content`, `header`, `footer`, `breadcrumb`, etc.). Run `drush config:get block.block.{name}` to enumerate placed blocks per region before swapping themes.

---

## Site Studio Sync Packages & Style Helpers

- **`cohesion_sync`** — Site Studio's config-package import/export module. Won't be used post-migration. Remove any sync packages stored in `docroot/sites/default/sync_packages/` (or wherever the site keeps them) once their contents have been ported.
- **`cohesion_style_helpers`** — provides "style helpers" (reusable style snippets applied to components). These map to either your CSS framework's component classes or, for one-off usage, inline utility composition.
- **`cohesion_style_guide`** — admin-facing style guide UI. Not migrated; remove with the module uninstall.
- **`cohesion_breakpoint_indicator`** — dev helper. Drop.
- **`sitestudio_data_transformers`** — bespoke Twig functions used inside Site Studio components. If any custom code components use these, port their logic to plain Twig or a custom Twig extension in the theme before disabling.
- **`sitestudio_page_builder`** — provides the Layout Canvas widget. Disabled only after all `layout_canvas` fields are removed.
- **`sitestudio_governance`** — workflow / publishing guards. Replace with core Content Moderation if those guards are still needed.

Site Studio "variable fields" (`cohesion_website_settings.cohesion_variable_field.*.yml`) are site-wide variables (colors, fonts, spacings) referenced by `$variable` syntax in Site Studio CSS. Port to your CSS framework's theme variables — see `@references/style-migration.md`.

---

## Acquia DAM Integration

If the site uses Acquia DAM via the `acquia_dam` module, the media library picker in Canvas works the same way — DAM-backed media types appear in the picker exactly like local media types. Two gotchas:

- Components declared with `$ref: json-schema-definitions://canvas.module/image` accept any media type whose source uses the `image` media source. If a DAM media bundle uses a non-image source, it won't be selectable by default — either include it via `hook_canvas_storable_prop_shape_alter()` or declare the prop with a custom shape that targets that bundle.
- Image styles applied to DAM-backed remote images may be delivered by the DAM CDN; the StaticPropSource expression may need adjustment if the URL comes from a different field than `field_media_image`. Inspect the media type at `/admin/structure/media/manage/{type}/fields`.

---

## Multilingual / Translations

Site Studio's `layout_canvas` field is typically configured as **not translatable** — the JSON contains keys that need to be translated separately via Drupal's interface translation system or via the Site Studio component content config translation.

Canvas's `component_tree` field can be either translatable or not. Recommended approach:

- Make the new `field_canvas_layout` translatable per bundle. Each translation stores its own component tree.
- During migration: for each translation that exists on the source `layout_canvas`, run the migration plugin against that translation. Add a `default_value` process step that calls `migration_lookup` on the source translation map.
- Where the same Site Studio component instance carried multiple language values, those collapse to one set of `inputs` per translated node. The migration row plugin runs once per (node, language) pair.

If the site is monolingual, ignore this section.

---

## Testing & Visual Regression

The risk in this migration is not "the new field doesn't render" — it's "the new render is subtly different and only some pages notice." Treat visual regression as a primary deliverable.

- **BackstopJS or Playwright** screenshot diff: capture the legacy Site Studio render of every distinct template/component combination before phase 5. Re-run after cutover and triage diffs.
- **Side-by-side comparison**: run two copies of the site (or two database states) and use a side-by-side iframe diff tool. The `field_uses_canvas` toggle (described in `@references/layout-canvas-migration.md`) makes this easy on a single instance.
- **Per-component preview**: the Canvas component admin provides an isolated preview for each enabled SDC at `/admin/appearance/component/{id}`. Useful when chasing down which component is responsible for a diff.
- **Lighthouse / page-weight comparison**: Site Studio injects per-page compiled CSS and a lot of inline styles. Canvas + a utility-first CSS framework should produce smaller, cacheable assets. Capture before/after baselines.

Add a kernel test per SDC that asserts:
- The component is registered (`Component::load('sdc.{provider}.{name}')` returns a non-NULL, enabled entity).
- A sample `ComponentTreeItemList::setValue([…])` with realistic inputs validates with no violations.

---

## Module Uninstall Sequence

Order matters — Site Studio submodules have hard dependencies on each other. Uninstall **from leaves down to roots**.

```bash
# 1. Confirm no content depends on Site Studio fields.
drush sql:query "SELECT COUNT(*) FROM node__layout_canvas WHERE layout_canvas_value IS NOT NULL;"
# Must be 0 before continuing. Same check for any other Site Studio storage fields.

# 2. Delete the field storage on each bundle that had layout_canvas.
drush field:delete layout_canvas --entity-type=node --bundle=page
drush field:delete layout_canvas --entity-type=node --bundle=landing_page

# 3. Uninstall in this order (each command should report success before moving on):
drush pmu cohesion_style_guide
drush pmu cohesion_breakpoint_indicator
drush pmu sitestudio_governance
drush pmu sitestudio_legacy_ckeditor
drush pmu sitestudio_page_builder
drush pmu sitestudio_data_transformers
drush pmu sitestudio_claro
drush pmu cohesion_templates
drush pmu cohesion_custom_styles
drush pmu cohesion_base_styles
drush pmu cohesion_style_helpers
drush pmu cohesion_elements
drush pmu cohesion_website_settings
drush pmu cohesion_sync
drush pmu cohesion
drush pmu sitestudio_gin   # if installed

# 4. Remove from composer.
composer remove acquia/cohesion

# 5. Export config.
drush cex

# 6. Delete leftover config files manually.
rm -rf config/{site}/{site-studio-config-dir}/
```

Each `pmu` will refuse if any module still depends on it — that's the safety net. Don't force with `--yes` until the depending modules are also queued.

If `cohesion` itself refuses to uninstall, the most common cause is a residual `cohesion_layout` field or `cohesion_layouts` storage field somewhere; `drush field:info` to enumerate.

---

## Performance & Caching Differences

Site Studio compiles per-component CSS into per-page `<style>` blocks at render time, plus a global compiled stylesheet. The result is large pages with heavy inline CSS.

Canvas + a utility-first CSS framework characteristics:
- Output is a single static CSS bundle (one file per theme), cacheable indefinitely.
- SDC libraries auto-attach only when the component is rendered, but their CSS is part of a deterministic library bundle (not per-page-compiled).
- Page composition is rendered via Twig + the `component_tree` field's render array. No JIT CSS compilation per request.

Expect lower TTFB and smaller page weight after migration. Re-baseline cache settings in `settings.php` (Site Studio recommends some aggressive cache rules that may no longer apply): inspect `$settings['cache']`, `$config['system.performance']`, and `services.yml` `http.response.debug_cacheability_headers`.

After uninstall, look for and remove:
- `$settings['cohesion_*']` lines in `settings.php`
- Apache/nginx rules referencing `/sites/default/files/cohesion/`
- The `sites/default/files/cohesion/` directory itself (after backing up — nothing in here should be authoritative post-migration)
