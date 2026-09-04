# Built-in Site Studio Elements → SDC Equivalents

Built-in elements are the core Site Studio elements available to all sites — not custom components. They appear in Layout Canvas JSON by their `uid` string. Each one needs an SDC equivalent (or a Canvas native block) for the page migration.

## Table of Contents

1. [How Built-in Elements Appear in Layout Canvas JSON](#layout-canvas-json-structure)
2. [Elements Replaceable by Canvas Native Blocks](#canvas-native-blocks)
3. [Elements Requiring SDCs](#elements-requiring-sdcs)
4. [Worked Examples](#worked-examples)

---

## Layout Canvas JSON Structure

Built-in elements appear in the `canvas` array alongside component instances:

```json
{
  "canvas": [
    {
      "type": "item",
      "uid": "heading",
      "uuid": "aaa-111-...",
      "parentUid": "root",
      "children": []
    },
    {
      "type": "container",
      "uid": "accordion-tabs-container",
      "uuid": "bbb-222-...",
      "parentUid": "root",
      "children": ["ccc-333-..."]
    }
  ],
  "model": {
    "aaa-111-...": {
      "settings": {
        "headingType": "h2",
        "headingText": "Welcome"
      }
    }
  }
}
```

The `model[uuid].settings` structure varies per element type and is different from custom component model data (which uses `machineName` keyed values).

---

## Canvas Native Blocks

These built-in elements map directly to Drupal's block system and do **not** need custom SDCs. In Drupal Canvas, place the corresponding block from the block library:

| Site Studio `uid` | Drupal Canvas equivalent |
|---|---|
| `drupal-block` | Place any core/contrib block via the block picker |
| `drupal-menu` | Place a menu block (Main navigation, Footer, etc.) |
| `breadcrumb` | Place the `system_breadcrumb_block` |
| `entity-browser` | Use Canvas's built-in media picker for media fields |
| `entity-reference` | Use Canvas's entity reference field type |

---

## Elements Requiring SDCs

Create these as SDCs in the theme or a shared module. Group them under a consistent SDC `group` (e.g. `My Components/Elements`) so they're easy to find in the Canvas sidebar.

### `heading`

Settings keys: `headingType` (h1–h6), `headingText`, alignment, color class

```yaml
# heading/heading.component.yml
name: Heading
group: My Components/Elements
props:
  type: object
  required: [text]
  properties:
    text:
      type: string
      title: Heading text
      examples: ['Section Title']
    level:
      type: string
      title: Heading level
      enum: [h1, h2, h3, h4, h5, h6]
    align:
      type: string
      title: Alignment
      enum: [left, center, right]
    color:
      type: string
      title: Color class
      examples: ['text-primary']
```

```twig
{# heading/heading.twig #}
{% set tag = level|default('h2') %}
<{{ tag }} {{ attributes.addClass([align ? 'text-' ~ align, color]) }}>
  {{- text -}}
</{{ tag }}>
```

### `paragraph` / `wysiwyg`

WYSIWYG content should be a **slot** in the SDC (or rendered as processed text):

```yaml
# rich-text/rich-text.component.yml
name: Rich Text
group: My Components/Elements
slots:
  content:
    title: Content
    description: Rich text / WYSIWYG content.
```

```twig
<div {{ attributes.addClass('prose max-w-none') }}>
  {% block content %}{% endblock %}
</div>
```

### `button`

Settings keys: `buttonText`, `buttonLink`, `buttonTarget`, `buttonType` (style variant)

```yaml
name: Button
group: My Components/Elements
props:
  type: object
  required: [text, href]
  properties:
    text:
      type: string
      title: Button text
    href:
      type: string
      title: URL
      format: uri-reference
    target:
      type: string
      title: Target
      enum: [_self, _blank]
    variant:
      type: string
      title: Variant
      enum: [primary, secondary, outline]
      examples: [primary]
```

```twig
<a href="{{ href }}"
   {{ target ? 'target="' ~ target ~ '"' }}
   {{ attributes.addClass(['btn', 'btn-' ~ (variant|default('primary'))]) }}>
  {{ text }}
</a>
```

### `link`

Simpler than button — inline text link:

```yaml
name: Link
group: My Components/Elements
props:
  type: object
  required: [text, href]
  properties:
    text:
      type: string
      title: Link text
    href:
      type: string
      title: URL
      format: uri-reference
    target:
      type: string
      enum: [_self, _blank]
```

### `image`

Settings keys: `imageEntityId` (media entity), `alt`, `imageLink`, `imageStyle`

Use Canvas's media `$ref` so the prop gets a media-library picker in the editor and stores as a media entity reference (not a URL). Canvas resolves `src`/`alt`/`width`/`height` at render time via the prop expression.

```yaml
name: Image
group: My Components/Elements
props:
  type: object
  required: [image]
  properties:
    image:
      title: Image
      $ref: json-schema-definitions://canvas.module/image
    href:
      type: string
      title: Link URL
      format: uri-reference
```

```twig
{% if href %}<a href="{{ href }}">{% endif %}
<img {{ attributes }} src="{{ image.src }}" alt="{{ image.alt|default('') }}"
     {% if image.width %}width="{{ image.width }}"{% endif %}
     {% if image.height %}height="{{ image.height }}"{% endif %}>
{% if href %}</a>{% endif %}
```

The `$ref` resolves to an object with `src`, `alt`, `width`, `height` at render time — write twig as if those were direct sub-properties of `image`. Storage uses an entity_reference to the media entity (see `@references/content-migration.md` for the StaticPropSource shape).

### `video`

```yaml
name: Video
group: My Components/Elements
props:
  type: object
  properties:
    src: { type: string, title: Video URL, format: uri-reference }
    poster: { type: string, title: Poster image, format: iri-reference }
    autoplay: { type: boolean }
    loop: { type: boolean }
    muted: { type: boolean }
```

### `youtube-video-embed`

```yaml
name: YouTube Video
group: My Components/Elements
props:
  type: object
  required: [video_id]
  properties:
    video_id: { type: string, title: YouTube video ID, examples: ['dQw4w9WgXcQ'] }
    title: { type: string, title: Accessible title }
```

```twig
<div {{ attributes.addClass('aspect-video') }}>
  <iframe class="w-full h-full"
    src="https://www.youtube.com/embed/{{ video_id }}"
    title="{{ title|default('YouTube video') }}" allowfullscreen></iframe>
</div>
```

### `blockquote`

```yaml
name: Blockquote
group: My Components/Elements
props:
  type: object
  required: [quote]
  properties:
    quote: { type: string, title: Quote text }
    citation: { type: string, title: Citation }
```

### `iframe`

```yaml
name: iFrame
group: My Components/Elements
props:
  type: object
  required: [src]
  properties:
    src: { type: string, title: iFrame URL, format: uri }
    title: { type: string, title: Accessible title }
    height: { type: integer, title: Height (px) }
```

### `accordion-tabs-container` + `accordion-tabs-item`

Container/item pattern — same slot approach as the custom accordion components:

```yaml
# accordion-container/accordion-container.component.yml
name: Accordion
group: My Components/Elements
slots:
  items:
    title: Accordion items
```

```yaml
# accordion-item/accordion-item.component.yml
name: Accordion Item
group: My Components/Elements
props:
  type: object
  required: [title]
  properties:
    title:
      type: string
      title: Item title
    open:
      type: boolean
      title: Open by default
slots:
  content:
    title: Item content
```

### `slider-container`

```yaml
name: Slider
group: My Components/Elements
props:
  type: object
  properties:
    autoplay:
      type: boolean
    speed:
      type: integer
      title: Transition speed (ms)
slots:
  slides:
    title: Slides
```

### `modal` + `modal-button`

```yaml
# modal/modal.component.yml
name: Modal
group: My Components/Elements
props:
  type: object
  required: [id]
  properties:
    id:
      type: string
      title: Modal ID (unique)
    trigger_text:
      type: string
      title: Trigger button text
slots:
  content:
    title: Modal content
```

### `read-more`

```yaml
name: Read More
group: My Components/Elements
props:
  type: object
  properties:
    label:
      type: string
      title: Expand button label
      examples: ['Read more']
    collapse_label:
      type: string
      title: Collapse button label
      examples: ['Read less']
slots:
  content:
    title: Expandable content
```

### `google-map`

```yaml
name: Google Map
group: My Components/Elements
props:
  type: object
  required: [lat, lng]
  properties:
    lat:
      type: number
      title: Latitude
    lng:
      type: number
      title: Longitude
    zoom:
      type: integer
      title: Zoom level
      examples: [12]
    api_key:
      type: string
      title: Google Maps API key
slots:
  markers:
    title: Map markers
```

### `video-background`

Container with a video playing behind slotted content:

```yaml
name: Video Background
group: My Components/Elements
props:
  type: object
  required: [src]
  properties:
    src:
      type: string
      title: Video file URL
      format: uri-reference
    poster:
      type: string
      title: Poster image
      format: iri-reference
slots:
  content:
    title: Overlay content
```

---

## Worked Examples

### Migrating a `heading` element from Layout Canvas JSON

**Source JSON:**
```json
{
  "uid": "heading",
  "uuid": "abc-123",
  "model": {
    "abc-123": {
      "settings": {
        "headingType": "h2",
        "headingText": "Our Story"
      }
    }
  }
}
```

**In Drupal Canvas (manual):** open `/canvas/editor/node/{nid}`, drag the Heading SDC, set:
- `text` = `"Our Story"`
- `level` = `"h2"`

**In storage** (one row in the `component_tree` field):

| column | value |
|---|---|
| uuid | `abc-123` |
| component_id | `sdc.my_components.heading` |
| parent_uuid | NULL |
| slot | NULL |
| inputs | `{"text": "Our Story", "level": "h2"}` |

### Migrating an `accordion-tabs-container` with items

**Source JSON canvas array (simplified):**
```json
[
  { "uid": "accordion-tabs-container", "uuid": "aaa", "parentUid": "root", "children": ["bbb", "ccc"] },
  { "uid": "accordion-tabs-item", "uuid": "bbb", "parentUid": "aaa" },
  { "uid": "accordion-tabs-item", "uuid": "ccc", "parentUid": "aaa" }
]
```

**In storage** (three sibling rows — Canvas is flat):

| delta | uuid | component_id | parent_uuid | slot |
|---|---|---|---|---|
| 0 | aaa | `sdc.my_components.accordion-container` | NULL | NULL |
| 1 | bbb | `sdc.my_components.accordion-item` | aaa | `items` |
| 2 | ccc | `sdc.my_components.accordion-item` | aaa | `items` |

**In Drupal Canvas (manual):** open `/canvas/editor/node/{nid}`, drag the Accordion SDC, then drag two Accordion Item SDCs into the parent's `items` slot. Item title goes in the `title` prop; item body goes in the `content` slot (which is itself a sibling row with `parent_uuid` = item uuid and `slot` = `content`).
