---
name: sitestudio-to-canvas-migration
description: Site Studio (Cohesion) to Drupal Canvas migration skill. Use when migrating or deprecating Acquia Site Studio, converting Cohesion components to SDCs, converting Site Studio styles to Tailwind CSS, migrating Layout Canvas pages to Drupal Canvas, writing migration scripts for Site Studio content, converting form.json to component.yml, converting Cohesion Twig templates, converting UI-created Site Studio components, migrating built-in Site Studio elements like heading/text/image/button/container, or replacing Cohesion base styles and custom styles with Tailwind utilities.
---

# Site Studio → Drupal Canvas Migration

Covers deprecating Acquia Site Studio (Cohesion) on a Drupal 11 codebase in favor of **Drupal Canvas**.

Target stack: **Drupal Canvas** (`drupal/canvas`) + **SDCs** + **Tailwind CSS v4** (via Vite) — swap the CSS framework/build tool references below if your project uses something else; the Canvas/SDC mechanics are independent of your CSS choice.

Related skills, if present in your project: an SDC-authoring skill, a Drupal config-management skill.

This skill captures the *mechanics* of the migration — the parts that are the same on any Drupal 11 + Site Studio + Canvas codebase. It deliberately has no knowledge of your specific site names, component inventory, or module paths. If you're repeating this migration across multiple sites/codebases, write a short project-specific overlay skill that references this one and lists your own component inventory, module paths, and environment gotchas — see the "Companion overlay skill" note at the bottom.

---

## Three Types of Site Studio Components

| Type | Source | Migration approach |
|------|--------|--------------------|
| **Built-in elements** | Site Studio module core (heading, text, image, button, container…) | Create equivalent SDCs per element, or map to Canvas native blocks |
| **Custom code components** | `docroot/modules/custom/{your_module}/custom_components/` | Convert `form.json` + `.html.twig` to SDC `component.yml` + `.twig` |
| **UI-created components** | `config/{site}/site_studio/cohesion_elements.cohesion_component.*.yml` (or `config/sitestudio/{site}/...` depending on how your project names the config directory) | Parse `json_values` JSON to extract fields, build SDC from scratch |

---

## Component Identity: Two ID Forms — Don't Mix Them

- **SDC plugin id** (used in Twig, code, `Component::load()` arguments, Canvas API requests): `{provider}:{component-name}` — e.g. `my_components:cta-card`
- **Component config entity id** (used in `component_tree` field storage, config files, Drush): `sdc.{provider}.{component-name}` — e.g. `sdc.my_components.cta-card`

Canvas requires a **Component config entity** (`canvas.component.sdc.{provider}.{name}.yml`) for every SDC before authors can place it. Canvas auto-creates these on cache rebuild for SDCs that pass the source's eligibility audit. Verify with `drush config:get canvas.component.sdc.{provider}.{name}`; if missing, visit `/admin/appearance/component/{id}/audit`.

---

## Built-in Site Studio Elements

These appear in Layout Canvas JSON by `uid`. Each needs an SDC equivalent (or Canvas native block) for the page migration:

| UID | Display name | SDC equivalent approach |
|-----|-------------|------------------------|
| `heading` | Heading | SDC with `level` enum, `text` string |
| `paragraph` | Paragraph / Text | SDC with `text` string (or slot for rich text) |
| `wysiwyg` | WYSIWYG Editor | SDC slot |
| `button` | Button | SDC with `text`, `href`, `target` |
| `link` | Link | SDC with `text`, `href` |
| `image` | Image | SDC with image object prop |
| `picture` | Picture | SDC with image object + sources |
| `video` | Video | SDC with media object prop |
| `iframe` | iFrame | SDC with `src`, `title` props |
| `youtube-video-embed` | YouTube Video | SDC with `video_id` string |
| `video-background` | Video Background | SDC container with `src` prop + slot |
| `blockquote` | Blockquote | SDC with `quote`, `citation` props |
| `read-more` | Read More | SDC with slot + `label` prop |
| `accordion-tabs-container` | Accordion/Tabs | SDC container with slot |
| `accordion-tabs-item` | Accordion/Tabs Item | SDC with `title` + slot |
| `slider-container` | Slider Container | SDC container with slot |
| `modal` | Modal | SDC with `trigger_text` + slot |
| `google-map` | Google Map | SDC with `lat`, `lng`, `zoom` props |
| `google-map-marker` | Google Map Marker | SDC with `lat`, `lng`, `label` props |
| `drupal-block` | Drupal Block | Canvas native block (no SDC needed) |
| `drupal-menu` | Drupal Menu | Canvas native block (no SDC needed) |
| `breadcrumb` | Breadcrumb | Canvas native block (no SDC needed) |
| `entity-browser` | Entity Browser | Canvas media picker (no SDC needed) |
| `entity-reference` | Entity Reference | Canvas entity ref (no SDC needed) |
| `inline-element` | Inline Element | HTML element wrapper SDC |
| `custom-element` | Custom Element | Map to specific SDC per usage |

---

## form.json / json_values Field Type → SDC Prop Mapping

| Site Studio type | SDC prop |
|---|---|
| `cohEntityBrowser` (media/image) | `$ref: json-schema-definitions://canvas.module/image` (do NOT use a plain object with src/alt — Canvas needs the ref to render the media library picker and to store an entity reference) |
| `cohTypeahead` / `form-link` | `string`, `format: uri-reference` |
| `cohTextBox` / `form-input` | `string` |
| `cohWysiwyg` / `form-wysiwyg` | slot (not a prop) |
| `checkboxToggle` | `boolean` |
| `cohSelect` with enum options | `string` with `enum` array |
| Component `dropzone` (canvas item) | slot |
| `cohSection` (form group container) | no prop — group logically in YAML |
| `cohNumberSlider` | `integer` or `number` |
| `cohColorPicker` | `string` (hex) or Tailwind class enum |
| `cohIconPicker` | `string` (icon name) |

Machine name conversion: kebab-case `link-to-page` → snake_case prop `link_to_page`

## Canvas Storage Model (One Sentence Each)

- The `component_tree` field stores a **flat list** of rows, one per component instance — no nested JSON.
- Each row has columns: `uuid`, `component_id` (dotted form), `component_version` (optional xxh64 hash), `parent_uuid`, `slot`, `inputs` (json), `label`.
- Children of a slot are **sibling rows** with `parent_uuid` set to the parent UUID and `slot` set to the parent's slot machine name.
- `inputs` is keyed by SDC prop name; values are either plain scalars or `{sourceType, value, expression}` StaticPropSource arrays (image/media refs use the latter).

See `@references/layout-canvas-migration.md` and `@references/content-migration.md` for full detail.

---

## Twig Conversion Cheatsheet

```twig
{{ attach_library('my_components/x') }}          {# → remove; SDC auto-attaches #}
{{ field.title }}                                 {# → {{ title }} #}
{{ field.description['#text'] }}                 {# → {% block description %}{% endblock %} #}
{{ field.link_to_page }}                          {# → {{ link_to_page }} #}
{% if field.round_image %}                        {# → {% if round_image %} #}
{{ renderContent(type, viewMode, id) }}           {# → <img src="{{ image.src }}" alt="{{ image.alt }}"> #}
{{ dropzones['any-uuid'] }}                       {# → {% block content %}{% endblock %} #}
class="coh-style-heading-xl"                      {# → Tailwind classes e.g. text-4xl font-bold #}
class="coh-container-boxed"                       {# → container mx-auto px-4 #}
```

---

## Key Drush Commands

```bash
drush cr                                              # Rebuild — triggers Canvas component discovery
drush cex                                             # Export config (writes to active config dir)
drush config:list --filter='canvas.component'         # List Canvas Component config entities
drush config:get canvas.component.sdc.{provider}.{name}  # Inspect one Component config entity
drush migrate:status --group=canvas_migration         # Check migration status
drush migrate:import ss_to_canvas_pages               # Run content migration
drush migrate:rollback ss_to_canvas_pages              # Roll back content migration
drush sql:query "SELECT bundle, COUNT(*) FROM node__layout_canvas WHERE layout_canvas_value IS NOT NULL GROUP BY bundle;"  # Layout Canvas inventory
```

## Canvas URLs

| Purpose | URL |
|---|---|
| Canvas dashboard | `/canvas` |
| Canvas editor for any entity | `/canvas/editor/{entity_type}/{entity}` |
| Component config entity admin | `/admin/appearance/component` |
| Canvas REST API base | `/canvas/api/v0/...` |

(Earlier `/xb/...` paths were Experience Builder pre-rename and are no longer present.)

---

## References

- `@references/component-migration.md` — Custom code component conversion (form.json → component.yml), step-by-step process, worked example, Component config entity registration
- `@references/ui-component-migration.md` — Converting UI-created components (parsing `json_values`, no PHP/Twig starting point)
- `@references/builtin-elements.md` — Built-in Site Studio element SDC equivalents and worked examples
- `@references/style-migration.md` — Tailwind v4 setup, design token extraction, Cohesion class → Tailwind mapping
- `@references/layout-canvas-migration.md` — Canvas flat storage model, Layout Canvas JSON structure, Canvas URLs, parallel-running strategy
- `@references/content-migration.md` — Migrate API plugin emitting flat ComponentTreeItem rows, media reference shape, validation
- `@references/migration-strategy.md` — Phasing, discovery, templates, text formats, base theme change, module uninstall order, visual regression

---

## Companion overlay skill

This skill is intentionally generic — no site names, component inventories, or module paths. If you're running this migration on a real codebase, add a short second skill in the same project (e.g. `{yourorg}-canvas-migration`) that:

1. States it builds on this skill for the general mechanics.
2. Lists your actual custom component inventory (with counts and locations).
3. Lists your actual UI-created component inventory per site/config directory.
4. Documents environment-specific gotchas you hit (build steps, DI/service quirks, config sync overrides, etc.).

Keep that overlay skill short — it should mostly be tables and links back here, not a restatement of the mechanics.
