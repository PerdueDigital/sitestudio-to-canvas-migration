# Component Migration: Site Studio → SDC

## Table of Contents

1. [File Structure Comparison](#file-structure-comparison)
2. [Step-by-Step Conversion Process](#step-by-step-conversion-process)
3. [form.json Field Type Reference](#formjson-field-type-reference)
4. [Twig Template Conversion](#twig-template-conversion)
5. [Libraries Migration](#libraries-migration)
6. [Worked Example: cta-card](#worked-example-cta-card)
7. [Registering the SDC with Canvas](#registering-the-sdc-with-canvas)

---

## File Structure Comparison

**Site Studio custom component:**
```
custom_components/cta_card/
├── cta_card.custom_component.yml   # name, category, description, template ref, form ref
├── form.json                        # Component form builder config (fields, dropzones)
├── cta_card.html.twig               # Template using field.xxx and renderContent()
└── cta_card.css                     # Component styles
```

**SDC equivalent:**
```
components/cta-card/
├── cta-card.component.yml   # JSON Schema props + slots, group, description
├── cta-card.twig            # Template using direct prop variables and slot blocks
└── cta-card.css             # Component styles (auto-attached by SDC)
```

Note: `custom_components/` uses underscores; SDC `components/` uses kebab-case directory and file names.

---

## Step-by-Step Conversion Process

1. **Read `{name}.custom_component.yml`** — note `name`, `category`, `description`
2. **Read `form.json`** — parse `componentForm` array and `model` map to identify all fields:
   - Each `componentForm` item has a `uid` (field type) and UUID
   - Each UUID in `model` has a `settings.machineName` — this becomes the SDC prop name
   - Canvas items in `canvas` array are dropzones → become SDC slots
3. **Map each field** using the type table in the main SKILL.md
4. **Create `component.yml`** with mapped props and slots
5. **Convert the Twig template** — replace `field.xxx` with direct prop variables, `renderContent()` with image markup, dropzones with slot blocks, `attach_library()` removed
6. **Handle CSS** — copy existing CSS file into SDC directory; convert Cohesion utility classes to Tailwind (or your CSS framework of choice)
7. **Handle JS** — copy existing JS into SDC directory; declare dependencies in `libraryOverrides`
8. **Update your module's `*.libraries.yml`** — remove the entry for this component once migrated
9. Clear caches: `drush cr`

---

## form.json Field Type Reference

### `cohEntityBrowser` (uid: `form-entity-browser`)

Media library picker. Use Canvas's media `$ref` so the prop renders a media picker in the editor and stores an entity_reference to the media entity (not a URL).

```json
"settings": {
  "type": "cohEntityBrowser",
  "machineName": "image",
  "options": { "entityType": "media", "bundles": { "image": true } }
}
```

→ SDC prop:
```yaml
image:
  title: Image
  $ref: json-schema-definitions://canvas.module/image
```

The `$ref` resolves to an object exposing `src`/`alt`/`width`/`height` at render time. Storage uses a StaticPropSource entity_reference to the media entity — see `@references/content-migration.md` for the shape. **Do not declare a plain object prop with `src`/`alt` sub-properties** unless you specifically want to bypass the media library and accept a raw URL.

For non-image media (PDF, video, audio), use the corresponding ref:
- video: `$ref: json-schema-definitions://canvas.module/video`
- generic media: write a prop with an `entity_reference` storage and configure target bundles via `hook_canvas_storable_prop_shape_alter()`

### `cohTextBox` (uid: `form-input`)

Plain text input. Maps to a `string` prop.

```json
"settings": { "type": "cohTextBox", "machineName": "title", "title": "Title" }
```

→ SDC prop:
```yaml
title:
  type: string
  title: Title
  examples:
    - 'Enter a title'
```

### `cohTypeahead` (uid: `form-link`)

Internal link autocomplete (node/media/file). Maps to a URI string prop.

```json
"settings": { "type": "cohTypeahead", "machineName": "link-to-page" }
```

→ SDC prop:
```yaml
link_to_page:
  type: string
  title: Link URL
  format: uri-reference
```

Note: kebab-case `machineName` becomes snake_case prop name.

### `cohWysiwyg` (uid: `form-wysiwyg`)

Rich text editor. In SDC, rich/structured content belongs in a **slot** rather than a string prop.

```json
"settings": { "type": "cohWysiwyg", "machineName": "description" }
```

→ SDC slot:
```yaml
slots:
  description:
    title: Description
    description: Rich text body content.
```

Access in template as `{% block description %}{% endblock %}`.

### `checkboxToggle` (uid: `form-checkbox-toggle`)

Boolean toggle. Maps to a `boolean` prop.

```json
"settings": { "type": "checkboxToggle", "machineName": "round-image" }
```

→ SDC prop:
```yaml
round_image:
  type: boolean
  title: Round Image
```

### `cohSection` (uid: `form-section`)

A visual grouping container in the Site Studio form builder. **Not a field** — it has no `machineName`. Group the child fields logically in `component.yml` without adding a prop for the container itself.

### Component `dropzone` (canvas item, type: `item`, uid: `component-drop-zone-placeholder`)

A region where other components can be nested. Becomes a **slot** in SDC.

The slot name should be descriptive. Use `content` as the default name, or name it after the dropzone title.

→ SDC slot:
```yaml
slots:
  content:
    title: Content
    description: Drop zone for child components.
```

Template: `{% block content %}{% endblock %}`

---

## Twig Template Conversion

### Full conversion rules

| Site Studio pattern | SDC equivalent |
|---|---|
| `{{ attach_library('my_components/x') }}` | Remove — SDC auto-attaches CSS/JS |
| `{{ field.title }}` | `{{ title }}` |
| `{{ field.link_text }}` | `{{ link_text }}` |
| `{{ field.link_to_page }}` | `{{ link_to_page }}` |
| `{{ field.description['#text'] }}` | `{% block description %}{% endblock %}` (slot) |
| `{% if field.round_image %}` | `{% if round_image %}` |
| `{{ renderContent(type, viewMode, id) }}` | `<img src="{{ image.src }}" alt="{{ image.alt }}">` |
| `{{ dropzones['any-uuid'] }}` | `{% block content %}{% endblock %}` |
| `class="coh-style-heading-xl"` | Tailwind classes e.g. `class="text-4xl font-bold leading-tight"` |
| `class="coh-container-boxed"` | `class="container mx-auto px-4"` |

### attributes variable

SDC always provides `{{ attributes }}`. Use it on the root element:

```twig
<div {{ attributes.addClass(['cta-card', round_image ? 'cta-card--round-image' : '']) }}>
```

### Context isolation

Always render with `with_context = false` when including child components:
```twig
{{ include('my_components:icon-card', { title: title }, with_context = false) }}
```

---

## Libraries Migration

After migrating each component, remove its entry from your module's `*.libraries.yml`.

**Before (libraries.yml):**
```yaml
cta_card:
  css:
    theme:
      custom_components/cta_card/cta_card.css: {}
```

**After:** Entry deleted. The SDC `cta-card/cta-card.css` is auto-attached.

Keep any libraries that provide genuine external JS dependencies (e.g. a slider library, a mapping library) as standalone entries and reference them from the relevant `component.yml` via `libraryOverrides`. Once all components are migrated, the libraries file should only contain those external-dependency libraries.

---

## Worked Example: cta-card

### Source: `custom_components/cta_card/form.json` fields
- `cohEntityBrowser` machineName `image` → image object prop
- `checkboxToggle` machineName `round-image` → `round_image` boolean prop
- `cohTextBox` machineName `title` → `title` string prop
- `cohWysiwyg` machineName `description` → `description` slot
- Group (`cohSection`): contains `cohTextBox` `link-text` + `cohTypeahead` `link-to-page`
- `cohTextBox` machineName `link-text` → `link_text` string prop
- `cohTypeahead` machineName `link-to-page` → `link_to_page` uri-reference prop

### Target: `components/cta-card/cta-card.component.yml`

```yaml
$schema: https://git.drupalcode.org/project/drupal/-/raw/HEAD/core/assets/schemas/v1/metadata.schema.json
name: CTA Card
description: Call-to-action card with image, title, body text, and link.
group: My Components/Cards
props:
  type: object
  required:
    - title
  properties:
    image:
      title: Image
      $ref: json-schema-definitions://canvas.module/image
    title:
      type: string
      title: Title
      examples:
        - 'Card Title'
    link_to_page:
      type: string
      title: Link URL
      format: uri-reference
    link_text:
      type: string
      title: Link Text
      examples:
        - 'Learn More'
    round_image:
      type: boolean
      title: Round Image
slots:
  description:
    title: Description
    description: Rich text body content.
```

### Target: `components/cta-card/cta-card.twig`

```twig
<div {{ attributes.addClass(['cta-card', round_image ? 'cta-card--round-image' : '']) }}>
  {% if image.src %}
    <img class="cta-card__image" src="{{ image.src }}" alt="{{ image.alt|default('') }}"
         {% if image.width %}width="{{ image.width }}"{% endif %}
         {% if image.height %}height="{{ image.height }}"{% endif %}>
  {% endif %}
  <div class="cta-card__content">
    {% if title %}
      <h3 class="cta-card__title">{{ title }}</h3>
    {% endif %}
    {% block description %}{% endblock %}
    {% if link_to_page %}
      <a class="cta-card__link" href="{{ link_to_page }}">{{ link_text }}</a>
    {% endif %}
  </div>
</div>
```

Note: Tailwind utility classes replace `coh-style-*` classes in the final version. Start with BEM class names preserved for easier CSS migration, then convert to Tailwind utilities.

---

## Registering the SDC with Canvas

After writing the SDC files, the component must be exposed to Canvas as a **Component config entity**. Canvas auto-creates one per discovered SDC on cache rebuild, but only if the SDC passes the source's eligibility audit.

```bash
# Rebuild discovery + create/update Component config entities.
drush cr

# Confirm the Component config entity exists.
drush config:get canvas.component.sdc.my_components.cta-card

# If absent, visit the components admin to see why it was rejected.
# /admin/appearance/component
#   → look for the component; click "Audit" to see ineligible reasons.
```

Common reasons an SDC won't be auto-registered:
- A prop's shape has no matching `StorablePropShape` (no Drupal field type stores that JSON shape). Either change the prop, or implement `hook_canvas_storable_prop_shape_alter()`.
- The SDC's `$schema` is missing or malformed.
- A required prop has no `examples`. Canvas uses examples to seed default values.

After registration, also export config so the Component config entity is checked in:

```bash
drush cex
# Look for new files matching canvas.component.sdc.* in your active config directory
```

The Component config entity id is **always** `sdc.{provider}.{component-name}` — the colon-separated SDC plugin id with dots. Use this id (not the plugin id) anywhere Canvas storage references a component (e.g., the `component_id` column of a `component_tree` field).
