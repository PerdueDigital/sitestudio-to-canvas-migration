# Layout Canvas Migration: Site Studio → Drupal Canvas

## Table of Contents

1. [Site Studio Layout Canvas Data Structure](#site-studio-layout-canvas-data-structure)
2. [Drupal Canvas Storage Model](#drupal-canvas-storage-model)
3. [Component Config Entities (SDC Registration)](#component-config-entities-sdc-registration)
4. [Component UID → SDC ID Mapping](#component-uid--sdc-id-mapping)
5. [Manual Page Re-creation Workflow](#manual-page-re-creation-workflow)
6. [Identifying Nodes with Layout Canvas](#identifying-nodes-with-layout-canvas)
7. [Extracting Field Values from Layout Canvas JSON](#extracting-field-values-from-layout-canvas-json)
8. [Canvas Admin & Edit URLs](#canvas-admin--edit-urls)
9. [Parallel-Running Strategy](#parallel-running-strategy)

---

## Site Studio Layout Canvas Data Structure

Layout Canvas content is stored on node fields as JSON. The field is typically `field_layout_canvas` or `layout_canvas`.

Top-level structure:

```json
{
  "canvas": [
    {
      "type": "item",
      "uid": "cpt_cta_card",
      "uuid": "a1b2c3d4-...",
      "parentUid": "root",
      "children": []
    },
    {
      "type": "item",
      "uid": "cpt_flip_card",
      "uuid": "b2c3d4e5-...",
      "parentUid": "root",
      "children": ["c3d4e5f6-..."]
    }
  ],
  "componentForm": [],
  "mapper": {},
  "model": {
    "a1b2c3d4-...": {
      "settings": { "title": "CTA Card" },
      "componentContentId": null,
      "componentUid": "cpt_cta_card",
      "model": {
        "image": { "entity": { "#entityType": "media", "#entityId": "42" } },
        "title": "Example title",
        "link-to-page": "/products",
        "link-text": "Shop Now",
        "round-image": false
      }
    }
  }
}
```

### Key fields

- `canvas[].uid` — Site Studio component machine name (`cpt_cta_card`, `cpt_flip_card`, or built-in slug)
- `canvas[].uuid` — instance UUID
- `canvas[].parentUid` — `"root"` for top-level, or the parent UUID for nested items
- `canvas[].children` — child UUIDs (Site Studio's redundant representation of the tree; authoritative tree comes from `parentUid`)
- `model[uuid].model` — field values for that instance
- `model[uuid].model.image.entity.#entityId` — media entity id (not a file id)
- `model[uuid].model['link-to-page']` — kebab-case keys mirror `machineName` in `form.json`

### Four UID namespaces in canvas JSON

| Namespace | Example uid | Source location |
|---|---|---|
| UI-created components | `cpt_cta_card`, `cpt_hero_banner` | `config/.../cohesion_elements.cohesion_component.*.yml` |
| Custom code components | `cta_card`, `flip_card` | `docroot/modules/custom/.../custom_components/{name}/` |
| Built-in elements | `heading`, `image`, `button`, `wysiwyg` | Site Studio module core — see `@references/builtin-elements.md` |
| Helpers | `hlp_{name}` | `cohesion_elements.cohesion_helper.*.yml` — expand to constituent components before migrating |

---

## Drupal Canvas Storage Model

Drupal Canvas stores page composition on a **content entity field of type `component_tree`** (e.g. `field_canvas_layout`). The field is **flat and relational** — one row per component instance — not a nested JSON document.

Each `ComponentTreeItem` row has these columns:

| Column | Required | Description |
|---|---|---|
| `uuid` | yes | UUID for this component instance |
| `component_id` | yes | Component config entity id, format `sdc.{provider}.{name}` (e.g. `sdc.my_components.cta-card`) |
| `component_version` | no | xxh64 hash of a specific Component version. Omit → use active version. |
| `parent_uuid` | no | `NULL` for root items; parent UUID for slot children |
| `slot` | no | `NULL` at root; otherwise the parent component's slot machine name |
| `inputs` | yes | JSON: prop values keyed by SDC prop name. Scalars or StaticPropSource arrays. |
| `label` | no | Optional author-facing label override |

Hierarchy is reconstructed from `parent_uuid` + `slot`. Sibling render order within a slot follows field delta order.

Source: `docroot/modules/contrib/canvas/src/Plugin/Field/FieldType/ComponentTreeItem.php` (schema definition).

### Why this matters for migration

- A Site Studio dropzone with N children becomes N additional **sibling rows** with `parent_uuid = {parent}` and `slot = {slot-name}`.
- The SDC prop id format used in storage is **dotted** (`sdc.my_components.cta-card`), not the colon form (`my_components:cta-card`) — the colon form only appears in SDC plugin ids in code/twig.
- `inputs` values for media-backed props are StaticPropSource entity references, not URL strings. See `@references/content-migration.md` for the exact shape.

---

## Component Config Entities (SDC Registration)

Canvas does **not** automatically expose every SDC to authors. Each SDC must have a corresponding **Component config entity** (`canvas.component.sdc.{provider}.{name}`) created and enabled before it can be placed on a page.

For SDCs discovered by `SingleDirectoryComponent` (i.e. any well-formed `*.component.yml` in a module's `components/` directory), Canvas creates the Component config entity automatically on cache rebuild — **provided** the SDC meets the source requirements (all props are storable as Canvas field types, etc.). If the audit fails, the SDC will appear in `/admin/appearance/component` as ineligible and won't be available.

Useful commands:

```bash
# List all Canvas Component config entities.
drush config:list --filter='canvas.component'

# Inspect one component's config.
drush config:get canvas.component.sdc.my_components.cta-card

# Audit a component (shows version, dependencies, ineligible reasons).
# UI: /admin/appearance/component/sdc.my_components.cta-card/audit

# After adding or editing an SDC, rebuild caches so discovery runs.
drush cr
```

If a component does not appear in the Canvas sidebar after `drush cr`:
1. Visit `/admin/appearance/component` — look for the component and any disabled/ineligible status
2. Visit the `/audit` link to see why — usually a prop with no Canvas StorablePropShape match
3. Either change the prop shape, or add a `hook_canvas_storable_prop_shape_alter()` mapping

---

## Component UID → SDC ID Mapping

In code and twig, SDC plugin ids use the colon form (`my_components:cta-card`). In Canvas storage (`component_id` column) the dotted form is used (`sdc.my_components.cta-card`).

| Site Studio component UID | SDC plugin id (code) | Component config entity id (storage) |
|---|---|---|
| `cta_card` | `my_components:cta-card` | `sdc.my_components.cta-card` |
| `flip_card` | `my_components:flip-card` | `sdc.my_components.flip-card` |

Extend this table with every custom code component in your project. UI-created Site Studio components (`cpt_*` in `cohesion_elements.cohesion_component.*.yml`) need individual SDC equivalents — see `@references/ui-component-migration.md`.

---

## Manual Page Re-creation Workflow

For a small number of pages, manual re-creation is the safest path:

1. Identify the source node and read its existing Layout Canvas JSON
2. Confirm every SDC needed exists as an enabled Component config entity (`/admin/appearance/component`)
3. Open the Canvas editor at **`/canvas/editor/node/{nid}`**
4. For each component in the canvas array (root first, then children):
   - Drag the matching SDC from the Canvas sidebar
   - Fill in props from `model[uuid].model` values
   - For media props: use Canvas's media picker — pick the entity that matches the source `#entityId`
5. For dropzone children: drag SDCs into the matching slot on the parent component
6. Save and publish via the Canvas toolbar
7. Visual-diff the new page against the live Site Studio render before flipping the node to use the new field

---

## Identifying Nodes with Layout Canvas

```bash
# Direct SQL — fastest for inventory.
drush sql:query "SELECT entity_id, bundle FROM node__layout_canvas WHERE layout_canvas_value IS NOT NULL;"

# Equivalent via entity query.
drush php-eval "
  \$nids = \Drupal::entityQuery('node')
    ->condition('layout_canvas', NULL, 'IS NOT NULL')
    ->accessCheck(FALSE)
    ->execute();
  print_r(\$nids);
"
```

Cross-reference with the entity bundle to scope migration batches.

---

## Extracting Field Values from Layout Canvas JSON

```php
$node = \Drupal\node\Entity\Node::load(42);
$layout_canvas = $node->get('layout_canvas')->value;
$data = json_decode($layout_canvas, TRUE);

foreach ($data['canvas'] as $item) {
  $uuid = $item['uuid'];
  $uid = $item['uid'];
  $model = $data['model'][$uuid]['model'] ?? [];
  echo "Component: $uid (uuid=$uuid)\n";
  print_r($model);
}
```

### Image field shape in source JSON

```json
"image": {
  "entity": {
    "#entityType": "media",
    "#entityId": "42",
    "#bundle": "image"
  }
}
```

The `#entityId` is the **media entity id**. In Canvas, this becomes a StaticPropSource entity_reference to the same media entity — Canvas resolves the URL/alt/width/height at render time. **Don't extract a URL and store it as a plain string** — that loses media management and breaks the prop's contract for SDCs declared with `$ref: json-schema-definitions://canvas.module/image`.

---

## Canvas Admin & Edit URLs

Routes are defined in `docroot/modules/contrib/canvas/canvas.routing.yml`. Verified against Canvas 1.x:

| Purpose | URL |
|---|---|
| Canvas dashboard | `/canvas` |
| Canvas editor for any content entity | `/canvas/editor/{entity_type}/{entity}` |
| Components admin (Component config entities) | `/admin/appearance/component` |
| Component audit (per component) | `/admin/appearance/component/{id}/audit` |
| Canvas REST API base | `/canvas/api/v0/...` |
| Layout endpoints | `/canvas/api/v0/layout/{entity_type}/{entity}` |
| Auto-save endpoints | `/canvas/api/v0/auto-saves/{entity_type}/{entity}` |
| Media upload | `/canvas/api/v0/media/{media_type}/upload` |

Clear an auto-save conflict before publishing:

```bash
# Get CSRF token first.
TOKEN=$(curl -s https://your-site.example/session/token)

# Delete the auto-save row for one entity.
curl -X DELETE "https://your-site.example/canvas/api/v0/auto-saves/node/2" \
  -H "X-CSRF-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  --cookie-jar cookies.txt --cookie cookies.txt
```

(The older `/xb/...` route prefix was Experience Builder — fully removed in the Canvas rename.)

---

## Parallel-Running Strategy

Both fields can coexist on a bundle during migration. Keep `layout_canvas` on the node alongside a new `field_canvas_layout` (Component Tree). Decide which renders via:

1. **Field display order in `core.entity_view_display.*.yml`** — temporarily hide Site Studio's field and show Canvas's, or vice versa. Toggle per-node by overriding view display via a custom preprocess.
2. **A per-bundle migration flag** — add a base field `field_uses_canvas` (boolean). In a preprocess hook, render `field_canvas_layout` when TRUE, otherwise `layout_canvas`. The migration sets the flag on successful import.
3. **Per-node feature flag** — flip individual nodes one at a time, validate, and only delete the Site Studio field after a full audit.

Once all migration is verified:
1. Mark the `layout_canvas` field deprecated
2. Run `drush field:delete layout_canvas --entity-type=node --bundle={bundle}` per bundle
3. Uninstall Site Studio modules in dependency order — see `@references/migration-strategy.md`
