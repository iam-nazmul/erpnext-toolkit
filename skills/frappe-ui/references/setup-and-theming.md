# frappe-ui Setup & Theming

## Installing into a Vue 3 project

The scaffolds (`bench add-frappe-ui`, `frappe-ui-starter`) come pre-wired — prefer them. Manual setup for an existing Vue 3 + Vite + Tailwind project:

```bash
npm install frappe-ui
```

```js
// main.js
import { createApp } from 'vue'
import { setConfig, frappeRequest, resourcesPlugin } from 'frappe-ui'
import App from './App.vue'

setConfig('resourceFetcher', frappeRequest)  // only needed with a Frappe backend
const app = createApp(App)
app.use(resourcesPlugin)                     // only needed for createResource & friends
app.mount('#app')
```

Components are imported individually (tree-shakeable) — there is no global component registration:

```js
import { Button, Dialog, FormControl } from 'frappe-ui'
```

Without a Frappe backend, skip `setConfig`/`resourcesPlugin` entirely and use only components + the preset.

## Tailwind preset

```js
// tailwind.config.js
module.exports = {
  presets: [require('frappe-ui/src/tailwind/preset')],
  content: [
    './index.html',
    './src/**/*.{vue,js,ts}',
    './node_modules/frappe-ui/src/components/**/*.{vue,js}', // required — or component styles get purged
  ],
}
```

Older releases exposed the preset at `frappe-ui/src/utils/tailwind.config`; check `node_modules/frappe-ui` if the require fails. Newer frappe-ui versions also ship a Vite plugin (`frappe-ui/vite`) that handles build integration (Lucide icon imports, build into a Frappe app's `www/`); the starter's `vite.config.js` shows the current wiring.

What the preset changes vs stock Tailwind:

- **Denser type scale**: `text-base` ≈ 14px, steps recalibrated for app UIs; paragraph sizes `text-p-xs`…`text-p-2xl` for prose.
- **Frappe gray palette** (gray-50…gray-900) tuned to match Desk and Frappe products.
- **Semantic tokens** (the important part, newer versions):
  - Text: `text-ink-gray-9` (headings) … `text-ink-gray-5` (muted), `text-ink-white`, `text-ink-blue-3`, `text-ink-red-4`
  - Backgrounds: `bg-surface-white`, `bg-surface-gray-1`…`bg-surface-gray-7`, `bg-surface-modal`, `bg-surface-selected`
  - Borders: `border-outline-gray-1`…`border-outline-gray-4`, `border-outline-white`

## Dark mode

Semantic tokens are CSS-variable-backed, so dark mode is a matter of toggling a `data-theme="dark"` attribute (or `dark` class, per version) on `<html>` — every `ink/surface/outline` class flips automatically. Raw palette classes (`bg-white`, `text-gray-800`) do **not** flip; that's why new code should use tokens. Products typically persist the choice:

```js
function toggleTheme() {
  const theme = document.documentElement.getAttribute('data-theme') === 'dark' ? 'light' : 'dark'
  document.documentElement.setAttribute('data-theme', theme)
  localStorage.setItem('theme', theme)
}
```

Check how the installed version detects theme (some export a `useTheme`/`theme` helper) before rolling your own.

## Icons

- `FeatherIcon` component: `<FeatherIcon name="x" class="h-4 w-4" />` — the classic set, used by `icon` props on Button/Dropdown.
- Newer versions favor **Lucide** icons via the Vite plugin: `import LucideAlertCircle from '~icons/lucide/alert-circle'` (unplugin-icons style). Match whichever the project already uses.
- Custom SVGs: inline them as Vue components; size with Tailwind `h-4 w-4` and color with `text-ink-*` so they theme correctly.

## The Frappe look, in rules of thumb

White/gray surfaces, `rounded` / `rounded-lg` corners, subtle `shadow-sm`, 1px `border-outline-gray-2` borders, small type (`text-base` body, `text-lg`/`text-xl` headings), medium-weight labels, generous whitespace, brand color used sparingly (primary buttons, links, focus rings). Grays carry the UI.
