# UI-Created Component Migration: Site Studio Config → SDC

UI-created components live entirely in config as `cohesion_elements.cohesion_component.*.yml` files. They have no PHP code or Twig files — all structure and fields are encoded in the `json_values` field as a JSON string.

## Table of Contents

1. [Reading json_values](#reading-json_values)
2. [Extracting Fields from componentForm](#extracting-fields-from-componentform)
3. [Migration Process](#migration-process)
4. [Helpers](#helpers)

---

## Reading json_values

Each component yml has a `json_values` key containing a multiline JSON string with these top-level keys:

```yaml
json_values: |
  {
    "canvas": [...],        # Visual canvas layout — how sub-elements are arranged
    "componentForm": [...], # Form fields shown to editors in the sidebar
    "mapper": {...},        # Maps canvas element UUIDs to form field values
    "model": {...},         # Default values for each field (keyed by UUID)
    "previewModel": {...},
    "variableFields": {...}
  }
```

To read a component's fields: parse the `json_values` JSON and inspect the `componentForm` array and `model` map.

### Reading from the command line

```bash
# Print componentForm field titles and types for a component
drush php-eval "
  \$config = \Drupal::config('cohesion_elements.cohesion_component.cpt_example');
  \$data = json_decode(\$config->get('json_values'), TRUE);
  foreach (\$data['model'] as \$uuid => \$item) {
    \$type = \$item['settings']['type'] ?? 'unknown';
    \$title = \$item['settings']['title'] ?? '-';
    \$machine = \$item['settings']['machineName'] ?? '-';
    if (\$machine !== '-') echo \"{$type}: {$title} (machineName: {$machine})\n\";
  }
"
```

---

## Extracting Fields from componentForm

Walk the `componentForm` array. Each item is a form field or container:

```json
{
  "type": "form-field",
  "uid": "form-input",         // field type
  "title": "Heading text",     // display label
  "uuid": "abc123...",         // key into model{}
  "parentUid": "root"          // "root" = top-level; a UUID = inside a form-section
}
```

Then look up `model[uuid].settings` for:
- `machineName` — becomes the SDC prop name (convert kebab-case → snake_case)
- `type` — determines the SDC prop type (see mapping table in main SKILL.md)
- `title` — becomes the prop `title` in `component.yml`

### Form field UIDs → SDC prop types

| `uid` | `type` in model | SDC |
|---|---|---|
| `form-input` | `cohTextBox` | `string` |
| `form-wysiwyg` | `cohWysiwyg` | slot |
| `form-entity-browser` | `cohEntityBrowser` | image object prop |
| `form-link` | `cohTypeahead` | `string`, `format: uri-reference` |
| `form-checkbox-toggle` | `checkboxToggle` | `boolean` |
| `form-select` | `cohSelect` | `string` with `enum` |
| `form-number-slider` | `cohNumberSlider` | `integer` |
| `form-color` | `cohColorPicker` | `string` (hex) |
| `form-icon` | `cohIconPicker` | `string` |
| `form-section` | `cohSection` | no prop (visual grouping only) |
| `form-tab-container` | — | no prop |
| `form-tab-item` | — | no prop |
| canvas `item` uid=`component-drop-zone-placeholder` | — | slot |

---

## Migration Process

For a UI-created component (no existing PHP/Twig):

1. **Find the yml**: `config/{site}/{site-studio-config-dir}/cohesion_elements.cohesion_component.{id}.yml`
2. **Parse `json_values`**: extract `componentForm` and `model`
3. **List all fields**: walk `model`, collect `machineName`, `type`, `title` for entries with a `machineName`
4. **Identify slots**: any canvas item with `uid: component-drop-zone-placeholder` becomes a slot
5. **Build `component.yml`**: map fields to props using the type table above
6. **Build the Twig template**: study the `canvas` array to understand the HTML structure and element nesting, then write equivalent markup
   - The canvas describes layout (containers, rows, columns) — translate to Tailwind-classed divs (or your CSS framework's equivalent)
   - Field values in canvas elements reference model UUIDs — map to prop variable names
7. **Extract styles**: inspect `mapper[uuid].settings.formDefinition` for style properties (padding, colors, border-radius) — translate to your CSS framework's utilities
8. **Verify against design**: compare rendered SDC output to the Site Studio component preview

Do a full inventory pass before converting anything — walk every `cohesion_elements.cohesion_component.*.yml` in your Site Studio config directory and group them by category (layout, basic/content, cards, sliders, accordion/tabs, dynamic/embed). Converting the highest-usage categories first (see the usage-histogram command in `@references/migration-strategy.md`) gets the most value for the least work; components used zero times in actual content can be skipped entirely.

---

## Helpers

Helpers are pre-composed combinations of components with preset field values. They appear in the Layout Canvas JSON using their helper UID (prefixed `hlp_`).

**Migration approach for helpers:**
- Helpers resolve to their constituent components — no separate SDC is needed
- When migrating a page that uses a helper, expand it to its component instances and migrate each component individually
- The preset field values in the helper's `json_values.model` carry the actual content

To inspect a helper's composition:
```bash
# Print canvas structure of a helper
drush php-eval "
  \$config = \Drupal::config('cohesion_elements.cohesion_helper.hlp_example');
  \$data = json_decode(\$config->get('json_values'), TRUE);
  foreach (\$data['canvas'] as \$item) {
    echo \$item['uid'] . ' (parent: ' . \$item['parentUid'] . \")\n\";
  }
"
```
