# Content Migration: Site Studio Layout Canvas → Drupal Canvas

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Canvas Storage Model You Must Match](#canvas-storage-model-you-must-match)
3. [Migration Module Structure](#migration-module-structure)
4. [Source Plugin](#source-plugin)
5. [Process Plugin: LayoutCanvasToComponentTree](#process-plugin-layoutcanvastocomponenttree)
6. [Migration YAML Config](#migration-yaml-config)
7. [Running Migrations](#running-migrations)
8. [Rollback and Re-run](#rollback-and-re-run)
9. [Media Entity ID Resolution](#media-entity-id-resolution)
10. [Validation After Import](#validation-after-import)

---

## Architecture Overview

If your codebase already has `migrate`, `migrate_plus`, and `migrate_tools` installed, use these to automate transition from Site Studio Layout Canvas JSON to a Drupal Canvas `component_tree` field. Otherwise, `composer require` them.

**Approach:**
1. Source: query nodes whose `layout_canvas` field has Site Studio JSON
2. Process: parse the Site Studio JSON, walk the tree, emit one row per component instance
3. Destination: update the same nodes — write a `component_tree` field value list

**Module location:** new `docroot/modules/custom/canvas_migration/` module.

---

## Canvas Storage Model You Must Match

Drupal Canvas stores page composition in a `component_tree` field type (`ComponentTreeItem`). **It is NOT a nested JSON document.** It is a **flat, ordered list of field-item rows**, each row being one component instance. Hierarchy is reconstructed at runtime from `parent_uuid` + `slot`.

Each row (column → meaning):

| Column | Required | Description |
|---|---|---|
| `uuid` | yes | Unique UUID for this component instance |
| `component_id` | yes | Component config entity ID, format `sdc.{provider}.{component-name}` (e.g. `sdc.my_components.cta-card`) |
| `component_version` | no | xxh64 hash of a specific Component config entity version. Omit to use the active version. |
| `parent_uuid` | no | `NULL` for top-level (root) items; the UUID of the parent component for nested items |
| `slot` | no | `NULL` at root; otherwise the slot machine name on the parent (must match an actual slot of that component's SDC) |
| `inputs` | yes | JSON: prop values. Either plain scalars for static values, or `{sourceType, value, expression}` shape for prop sources |
| `label` | no | Optional content-author-facing label |

Critical implications for migration:
- **Slot children are sibling rows**, not nested arrays. Each child row carries its parent's UUID in `parent_uuid` and the slot name in `slot`.
- **Item ordering inside the field** determines render order within a slot (siblings with same parent_uuid+slot render in field delta order).
- **`component_id` uses dots**: SDC plugin id `my_components:cta-card` → Component config entity id `sdc.my_components.cta-card`.
- **Component config entity must exist** for every `component_id` you reference. Canvas auto-creates Component config entities for discovered SDCs that meet source requirements; verify with `drush config:get canvas.component.sdc.{provider}.{name}`.
- **`inputs` is keyed by prop name** (the SDC prop machine name). Scalar values are stored as-is; richer values (entity references, link items, image media references) take the StaticPropSource shape `{sourceType, value, expression}`.

### Image and media inputs

A media-backed prop (declared in `*.component.yml` with `$ref: json-schema-definitions://canvas.module/image`) is stored as a `StaticPropSource` whose `sourceType` is an entity_reference field item targeting a media entity:

```json
"image": {
  "sourceType": "static:field_item:entity_reference",
  "value": { "target_id": 42 },
  "expression": "ℹ︎␜entity:media␝field_media_image␞␟{src↝entity␜␜entity:file␝uri␞0␟value,alt↠alt,width↠width,height↠height}"
}
```

The `target_id` is the **media entity id**, not a file id. Canvas resolves the URL/alt/width/height at render time via the expression. **Do not store a raw `{src, alt}` object when a media picker is wanted** — that locks the value to a URL and bypasses media management.

### Link inputs

A `uri-reference` prop is typically stored as:

```json
"href": {
  "sourceType": "static:field_item:uri",
  "value": "https://example.com",
  "expression": "ℹ︎uri␟value"
}
```

Plain scalar strings also work for unsourced static values; Canvas wraps them as needed.

---

## Migration Module Structure

```
docroot/modules/custom/canvas_migration/
├── canvas_migration.info.yml
├── config/
│   └── install/
│       ├── migrate_plus.migration_group.canvas_migration.yml
│       └── migrate_plus.migration.ss_to_canvas_pages.yml
└── src/
    └── Plugin/
        └── migrate/
            └── process/
                └── LayoutCanvasToComponentTree.php
```

### `canvas_migration.info.yml`

```yaml
name: 'Canvas Migration'
type: module
description: 'Migrates Site Studio Layout Canvas content to Drupal Canvas.'
core_version_requirement: ^11
dependencies:
  - drupal:migrate
  - migrate_plus:migrate_plus
  - drupal:node
  - canvas:canvas
```

---

## Source Plugin

Use the built-in `content_entity:node` source from `migrate_plus`, filtered to nodes with Layout Canvas JSON populated. If `content_entity` filtering by null-ness is unreliable, fall back to a SQL source:

```yaml
source:
  plugin: sql
  query: |
    SELECT n.nid, n.vid, n.type, lc.layout_canvas_value
    FROM node_field_data n
    INNER JOIN node__layout_canvas lc
      ON lc.entity_id = n.nid AND lc.bundle = n.type
    WHERE lc.layout_canvas_value IS NOT NULL
      AND lc.bundle IN ('page', 'landing_page')
  ids:
    nid:
      type: integer
  database_state_key: default
```

---

## Process Plugin: LayoutCanvasToComponentTree

The process plugin returns an **ordered list of associative arrays**, one per component instance. The list maps onto `ComponentTreeItemList::setValue($items)`.

**File:** `src/Plugin/migrate/process/LayoutCanvasToComponentTree.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\canvas_migration\Plugin\migrate\process;

use Drupal\canvas\Entity\Component;
use Drupal\migrate\Attribute\MigrateProcess;
use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\MigrateSkipRowException;
use Drupal\migrate\ProcessPluginBase;
use Drupal\migrate\Row;

/**
 * Transforms Site Studio Layout Canvas JSON into Canvas ComponentTreeItem rows.
 *
 * @usage:
 *   plugin: layout_canvas_to_component_tree
 *   source: layout_canvas_value
 */
#[MigrateProcess('layout_canvas_to_component_tree')]
final class LayoutCanvasToComponentTree extends ProcessPluginBase {

  /**
   * Maps Site Studio component UID → SDC plugin id (provider:name, kebab-case).
   *
   * The Component config entity id is computed from this as
   * `sdc.{provider}.{name}`. Populate this with your actual component
   * inventory — it's intentionally a small illustrative set here.
   */
  private const COMPONENT_MAP = [
    // Custom code components (examples — replace with your real inventory).
    'cta_card'  => 'my_components:cta-card',
    'flip_card' => 'my_components:flip-card',

    // Built-in Site Studio elements → SDCs.
    'heading'             => 'my_components:heading',
    'paragraph'           => 'my_components:rich-text',
    'wysiwyg'             => 'my_components:rich-text',
    'button'              => 'my_components:button',
    'link'                => 'my_components:link',
    'image'               => 'my_components:image',
    'video'               => 'my_components:video',
    'youtube-video-embed' => 'my_components:youtube-video',
    'blockquote'          => 'my_components:blockquote',
    'iframe'              => 'my_components:iframe',
    'accordion-tabs-container' => 'my_components:accordion-container',
    'accordion-tabs-item' => 'my_components:accordion-item',
    'slider-container'    => 'my_components:slider',
    'modal'               => 'my_components:modal',
    'read-more'           => 'my_components:read-more',
    'google-map'          => 'my_components:google-map',
    'video-background'    => 'my_components:video-background',
  ];

  /**
   * Per-component prop name remapping (Site Studio key → SDC prop name).
   */
  private const PROP_MAP = [
    'cta_card' => [
      'title' => 'title',
      'link-to-page' => 'href',
      'link-text' => 'link_text',
      'round-image' => 'round_image',
      'image' => 'image',
    ],
    'heading' => [
      'headingText' => 'text',
      'headingType' => 'level',
    ],
    'button' => [
      'buttonText' => 'text',
      'buttonLink' => 'href',
      'buttonTarget' => 'target',
      'buttonType' => 'variant',
    ],
    // Add more per migrated component …
  ];

  public function transform(
    mixed $value,
    MigrateExecutableInterface $migrate_executable,
    Row $row,
    string $destination_property,
  ): mixed {
    if (empty($value)) {
      throw new MigrateSkipRowException('Empty layout_canvas value.');
    }

    $data = json_decode((string) $value, TRUE);
    if (!is_array($data) || empty($data['canvas'])) {
      throw new MigrateSkipRowException('Malformed layout_canvas JSON.');
    }

    $items = [];
    foreach ($data['canvas'] as $node) {
      $row_item = $this->buildRow($node, $data, $migrate_executable);
      if ($row_item !== NULL) {
        $items[] = $row_item;
      }
    }

    return $items;
  }

  /**
   * Builds one ComponentTreeItem row from one canvas node.
   */
  private function buildRow(array $node, array $data, MigrateExecutableInterface $exec): ?array {
    $uid   = $node['uid'] ?? '';
    $uuid  = $node['uuid'] ?? '';
    $pUid  = $node['parentUid'] ?? 'root';

    $sdcId = self::COMPONENT_MAP[$uid] ?? NULL;
    if (!$sdcId) {
      $exec->saveMessage("Unmapped Site Studio uid: $uid (uuid $uuid) — skipped.");
      return NULL;
    }

    $componentConfigId = $this->sdcIdToConfigEntityId($sdcId);

    // Verify the Component config entity exists & is enabled.
    $component = Component::load($componentConfigId);
    if (!$component || !$component->status()) {
      $exec->saveMessage("Component config entity $componentConfigId missing or disabled — skipped.");
      return NULL;
    }

    $model = $data['model'][$uuid]['model'] ?? [];

    $row = [
      'uuid'         => $uuid,
      'component_id' => $componentConfigId,
      // Omit component_version → Canvas uses the active version.
      'inputs'       => $this->transformInputs($uid, $model),
    ];

    if ($pUid !== 'root') {
      // Site Studio nests via parentUid + an implicit dropzone. Canvas needs a
      // named slot. We default to 'content'; map per-component if your SDC
      // uses a different slot name (e.g. 'items' for accordion, 'slides' for
      // slider — set $slot explicitly here based on the parent uid).
      $row['parent_uuid'] = $pUid;
      $row['slot'] = $this->slotForParent($data['canvas'], $pUid);
    }

    return $row;
  }

  /**
   * Translates SDC plugin id (provider:name) → Component config entity id.
   */
  private function sdcIdToConfigEntityId(string $sdcPluginId): string {
    \assert(str_contains($sdcPluginId, ':'));
    return 'sdc.' . str_replace(':', '.', $sdcPluginId);
  }

  /**
   * Builds the `inputs` map for one component instance.
   */
  private function transformInputs(string $uid, array $model): array {
    $inputs = [];
    $map = self::PROP_MAP[$uid] ?? [];

    foreach ($model as $key => $value) {
      $propName = $map[$key] ?? str_replace('-', '_', $key);

      // Media (image) reference → StaticPropSource with media entity reference.
      if (is_array($value) && isset($value['entity']['#entityType'], $value['entity']['#entityId'])
        && $value['entity']['#entityType'] === 'media'
      ) {
        $inputs[$propName] = $this->mediaRefInput((int) $value['entity']['#entityId']);
        continue;
      }

      // WYSIWYG / rich text → emit as plain string; SDC slot rendering happens
      // via a separate sibling row if the component uses a slot.
      if (is_array($value) && isset($value['text'])) {
        $inputs[$propName] = (string) $value['text'];
        continue;
      }

      // Link object {url, text, target}.
      if (is_array($value) && isset($value['url'])) {
        $inputs[$propName] = [
          'sourceType' => 'static:field_item:uri',
          'value' => $value['url'],
          'expression' => 'ℹ︎uri␟value',
        ];
        continue;
      }

      // Plain scalar.
      $inputs[$propName] = $value;
    }

    return $inputs;
  }

  /**
   * Returns the StaticPropSource shape for a media-backed image prop.
   *
   * Assumes the prop is declared in the SDC with:
   *   $ref: json-schema-definitions://canvas.module/image
   * and that the target media bundle uses field_media_image.
   */
  private function mediaRefInput(int $mediaId): array {
    return [
      'sourceType' => 'static:field_item:entity_reference',
      'value' => ['target_id' => $mediaId],
      'expression' => 'ℹ︎␜entity:media␝field_media_image␞␟{src↝entity␜␜entity:file␝uri␞0␟value,alt↠alt,width↠width,height↠height}',
    ];
  }

  /**
   * Returns the slot machine name to place children of $parentUuid into.
   *
   * Map per-parent-component because each SDC declares its own slot names.
   */
  private function slotForParent(array $canvas, string $parentUuid): string {
    foreach ($canvas as $item) {
      if (($item['uuid'] ?? '') === $parentUuid) {
        return match ($item['uid'] ?? '') {
          'accordion-tabs-container' => 'items',
          'slider-container'         => 'slides',
          'modal'                    => 'content',
          'read-more'                => 'content',
          'video-background'         => 'content',
          default                    => 'content',
        };
      }
    }
    return 'content';
  }

}
```

### Notes on the plugin design

- **Walks the canvas array in source order** so siblings end up in the right field delta order.
- **No recursion needed**: Canvas storage is flat, so the migration can be flat too. Each canvas node becomes one row.
- **`parent_uuid` references the same UUIDs that exist in the Site Studio source** — fine to reuse them. They just need to be unique within the destination field.
- **`label` is omitted** — leave NULL so the Component config entity's label is shown to authors.
- **WYSIWYG text** is passed as a scalar input. If a target SDC uses a slot for body text instead, your migration should emit a second sibling row representing a rich-text SDC, with `parent_uuid` set to the parent's UUID and `slot` set appropriately.
- **Site Studio helpers (`hlp_*`)** are not handled here — pre-expand them into their constituent components in a preprocessing step, or extend `COMPONENT_MAP` to alias each helper to its principal SDC.

---

## Migration YAML Config

### `config/install/migrate_plus.migration_group.canvas_migration.yml`

```yaml
id: canvas_migration
label: 'Canvas Migration'
description: 'Migrates Site Studio Layout Canvas pages to Drupal Canvas.'
```

### `config/install/migrate_plus.migration.ss_to_canvas_pages.yml`

```yaml
id: ss_to_canvas_pages
label: 'Migrate Site Studio Layout Canvas to Drupal Canvas'
migration_group: canvas_migration

source:
  plugin: content_entity:node
  bundle:
    - page
    - landing_page

process:
  nid: nid
  vid: vid
  type: type

  # Transform layout_canvas JSON to a flat list of ComponentTreeItem rows.
  # IMPORTANT: the destination field must be a `component_tree` field type
  # already present on the node bundle (e.g. field_canvas_layout). Add it via
  # config before running the migration.
  field_canvas_layout:
    plugin: layout_canvas_to_component_tree
    source: layout_canvas/0/value

destination:
  plugin: entity:node
  overwrite_properties:
    - field_canvas_layout
  translations: false

migration_dependencies:
  required:
    - image_to_media   # ensures media entity IDs referenced in the JSON exist
```

The destination `field_canvas_layout` is a `component_tree` field — add it to each migrated bundle via config before running. Migrate writes the full set of items per row; existing values are replaced.

---

## Running Migrations

```bash
# Install module.
drush en canvas_migration

# Import migration config.
drush cim --partial --source=docroot/modules/custom/canvas_migration/config/install

# Smoke test on one node only (limit + idlist).
drush migrate:import ss_to_canvas_pages --limit=1 --idlist=42 -v

# Full run with feedback.
drush migrate:import ss_to_canvas_pages --feedback=10

# Inspect messages from this migration.
drush migrate:messages ss_to_canvas_pages
```

The Migrate API does not have a true `--dry-run`; use `--limit=1` against a known node id and review the saved field manually instead.

---

## Rollback and Re-run

```bash
# Roll back all migrated rows.
drush migrate:rollback ss_to_canvas_pages

# Reset a stalled migration (status stuck on 'Importing').
drush migrate:reset-status ss_to_canvas_pages

# Re-process previously migrated rows after editing the process plugin.
drush migrate:import ss_to_canvas_pages --update
```

Rollback removes the **migration map row**, not the field value. To wipe the Canvas field as well, either:
- `migrate:rollback` then re-import (recommended), or
- For one-off resets: clear the field with a PHP-eval script before re-importing.

---

## Media Entity ID Resolution

Site Studio `cohEntityBrowser` fields store `model[uuid].model.{field}.entity.#entityId` — the **media** entity id. No file id resolution is necessary; Canvas resolves media → file → URL via the `expression` on the StaticPropSource at render time.

If the source database is from a different environment and media IDs differ, declare:

```yaml
migration_dependencies:
  required:
    - image_to_media
```

…and let an existing image/media migration map translate old → new media IDs (extend the process plugin to call `migration_lookup` against `image_to_media` for each `#entityId`).

---

## Validation After Import

After a successful run, validate one node end-to-end:

```bash
# 1. Confirm field rows were written.
drush sql:query "SELECT entity_id, delta, component_id, parent_uuid, slot FROM node__field_canvas_layout WHERE entity_id = 42 ORDER BY delta;"

# 2. Confirm Canvas can render it (load the editor).
#    Visit /canvas/editor/node/42 in the browser.

# 3. Run the field's typed-data validator from PHP.
drush php-eval "
  \$n = \Drupal\node\Entity\Node::load(42);
  \$v = \$n->get('field_canvas_layout')->validate();
  foreach (\$v as \$violation) {
    echo \$violation->getPropertyPath() . ': ' . \$violation->getMessage() . PHP_EOL;
  }
"
```

The third step exercises Canvas's `ComponentTreeStructure` + `ValidComponentTreeItem` constraints, which catch invalid `parent_uuid`/`slot` combinations, missing required props, and unknown `component_id` values.
