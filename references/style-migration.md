# Style Migration: Site Studio → Tailwind CSS v4

## Table of Contents

1. [Tailwind v4 Setup](#tailwind-v4-setup)
2. [Design Token Extraction](#design-token-extraction)
3. [Base Styles Migration](#base-styles-migration)
4. [Custom Styles Migration](#custom-styles-migration)
5. [Component-Level Styles](#component-level-styles)
6. [Cohesion Class Reference](#cohesion-class-reference)

Substitute your project's actual CSS framework/build tool if it isn't Tailwind v4 + Vite — the extraction *process* (pull design intent out of Cohesion config, express it as reusable tokens/utilities) is the transferable part.

---

## Tailwind v4 Setup

**Directory:** `docroot/themes/custom/{your_theme}/`

### `package.json`

```json
{
  "name": "your_theme",
  "private": true,
  "scripts": {
    "build": "vite build",
    "dev": "vite"
  },
  "devDependencies": {
    "@tailwindcss/vite": "^4.0.0",
    "tailwindcss": "^4.0.0",
    "vite": "^5.0.0"
  }
}
```

### `vite.config.js`

```js
import { defineConfig } from 'vite';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [tailwindcss()],
  build: {
    outDir: 'dist',
    rollupOptions: {
      input: 'src/css/main.css',
      output: {
        assetFileNames: 'css/[name][extname]',
      },
    },
  },
});
```

### `src/css/main.css`

```css
@import "tailwindcss";

@theme {
  /* Design tokens go here — populated from Site Studio color/typography config */
}

@layer base {
  /* Base element styles migrated from cohesion_base_styles */
}

@layer components {
  /* Reusable component patterns migrated from cohesion_custom_styles */
}
```

### `{your_theme}.libraries.yml`

```yaml
global:
  css:
    theme:
      dist/css/main.css: {}
```

### Build commands

```bash
cd docroot/themes/custom/{your_theme}
npm install
npm run build        # production build
npm run dev          # watch mode during development
```

---

## Design Token Extraction

Site Studio stores design tokens in `config/{site}/{site-studio-config-dir}/` as:
- `cohesion_website_settings.cohesion_color.*.yml` — color palette
- `cohesion_website_settings.cohesion_font_stack.*.yml` — font stacks
- `cohesion_website_settings.cohesion_website_settings.*.yml` — global settings

### Color extraction

Each color yml file contains a `value` key with color data:

```yaml
# cohesion_website_settings.cohesion_color.brand-primary.yml
id: brand-primary
label: 'Brand Primary'
value:
  value: '#003087'
  opacity: 100
```

→ Tailwind v4 CSS variable in `@theme`:

```css
@theme {
  --color-brand-primary: #003087;
  --color-white: #ffffff;
  /* ... one entry per cohesion_color yml */
}
```

Use these as Tailwind utilities: `text-brand-primary`, `bg-brand-primary`, `border-brand-primary`.

### Font stack extraction

```yaml
# cohesion_website_settings.cohesion_font_stack.primary.yml
label: 'Primary'
value:
  fontFamily: '"Your Brand Font", sans-serif'
```

→ Tailwind v4:

```css
@theme {
  --font-primary: "Your Brand Font", sans-serif;
}
```

Use as: `font-primary`

---

## Base Styles Migration

Site Studio base styles live in:
`config/{site}/{site-studio-config-dir}/cohesion_base_styles.cohesion_base_styles.*.yml`

Common base style entities:
- `article` — body text, paragraphs
- `heading` — h1–h6 defaults
- `table` — table element styles
- `form` — input, label, select defaults
- `list` — ul, ol, li defaults
- `blockquote` — blockquote element

Each yml has a `json_values` key with the CSS configuration encoded as JSON.

**Migration approach:** Extract the visual intent (font size, weight, color, spacing) and write equivalent Tailwind rules in `@layer base`:

```css
@layer base {
  h1 {
    @apply text-4xl font-bold leading-tight text-brand-primary;
  }
  h2 {
    @apply text-3xl font-semibold leading-snug;
  }
  h3 {
    @apply text-2xl font-semibold;
  }
  p {
    @apply text-base leading-relaxed;
  }
  a {
    @apply text-brand-primary underline hover:no-underline;
  }
}
```

### Cohesion breakpoints → Tailwind screens

Site Studio breakpoints (typically `xs`, `sm`, `md`, `lg`, `xl`) map to Tailwind responsive prefixes. Verify the exact pixel values in the Site Studio settings and match them in `@theme` if they differ from Tailwind defaults:

```css
@theme {
  --breakpoint-sm: 576px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px;
}
```

---

## Custom Styles Migration

Site Studio custom styles live in:
`config/{site}/{site-studio-config-dir}/cohesion_custom_styles.cohesion_custom_style.*.yml`

These are reusable CSS classes editors applied to components via the style picker (e.g., `coh-style-button-primary-link`, `coh-style-heading-xl`).

**Migration approach:**

1. Identify which `coh-style-*` class names appear in component Twig templates
2. For each one, find the corresponding custom style yml and read the CSS intent from `json_values`
3. Either:
   - **Replace inline**: Swap the class for Tailwind utilities directly on the markup element
   - **Create a component class**: If the style is reused across many components, add it to `@layer components`

```css
@layer components {
  .btn-primary {
    @apply inline-flex items-center px-6 py-3 bg-brand-primary text-white font-semibold rounded-sm hover:bg-blue-900 transition-colors;
  }
}
```

Then in Twig: `class="btn-primary"` instead of `class="coh-style-button-primary-link"`.

### Common cohesion class mappings

| Cohesion class | Tailwind equivalent |
|---|---|
| `coh-style-heading-xl` | `text-4xl font-bold leading-tight` |
| `coh-style-heading-lg` | `text-3xl font-semibold` |
| `coh-style-button-primary-link` | `btn-primary` (component class) or inline utilities |
| `coh-container-boxed` | `container mx-auto px-4` |
| `coh-style-p-t-medium` | `pt-8` |
| `coh-style-p-b-medium` | `pb-8` |
| `coh-wysiwyg` | `prose` (if using @tailwindcss/typography) |

---

## Component-Level Styles

Each Site Studio custom component has its own CSS file (e.g., `cta_card.css`). When migrating a component to an SDC:

**Option A — Preserve CSS, convert incrementally:**
Copy the existing CSS into the SDC directory as `{name}.css`. The SDC auto-attaches it. Then refactor class-by-class to Tailwind over time.

**Option B — Convert immediately:**
Replace the CSS file contents with Tailwind `@apply` directives or delete it entirely and write Tailwind utility classes directly on the template markup.

Example converting `cta_card.css` BEM classes to Tailwind on the markup:

```twig
{# Before: relying on cta_card.css #}
<div class="cta-card cta-card--round-image">
  <img class="cta-card__image">
  <div class="cta-card__content">
    <h3 class="cta-card__title">{{ title }}</h3>
  </div>
</div>

{# After: Tailwind utilities inline #}
<div class="flex flex-col bg-white rounded-lg shadow-md overflow-hidden
            {{ round_image ? 'rounded-full' : '' }}">
  <img class="w-full object-cover aspect-video">
  <div class="p-6 flex flex-col gap-4">
    <h3 class="text-xl font-semibold text-brand-primary">{{ title }}</h3>
  </div>
</div>
```

---

## Cohesion Class Reference

This is a worked example set from one real migration — expect your own `coh-style-*`/BEM class names to differ, but the pattern (inspect Cohesion's compiled CSS intent, then write the Tailwind equivalent) is the same:

| Cohesion class | Context | Tailwind |
|---|---|---|
| `coh-style-heading-xl` | hero banner h1 | `text-5xl font-bold leading-none` |
| `coh-container-boxed` | hero banner content wrapper | `container mx-auto px-6 max-w-7xl` |
| `cta-card__link coh-style-button-primary-link no-icon` | cta-card link | `btn-primary no-underline` |
| `banner` | hero banner root | `relative w-full overflow-hidden` |
| `banner-content` | hero banner overlay | `absolute inset-0 flex items-center` |
| `banner-text` | hero banner text block | `max-w-2xl text-white` |
| `flip-card` | flip-card root | `[perspective:1000px] w-full h-64` |
| `flip-card-inner` | flip-card inner | `relative w-full h-full transition-transform duration-700 [transform-style:preserve-3d]` |
| `flip-card-front` | flip-card front face | `absolute inset-0 [backface-visibility:hidden]` |
| `flip-card-back` | flip-card back face | `absolute inset-0 [backface-visibility:hidden] [transform:rotateY(180deg)]` |
