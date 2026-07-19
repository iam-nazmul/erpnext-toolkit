---
name: frappe-ui
description: Use the frappe-ui Vue 3 component library — Button, FormControl, Dialog, ListView, TextEditor, Autocomplete, toasts, and the createResource data layer — with its Tailwind preset, semantic design tokens, and dark mode. Use this skill whenever the user is working with frappe-ui components or props, asks "which frappe-ui component for X", needs theming/dark-mode/design tokens (ink/surface/outline classes), wants frappe-ui utilities/composables (call, debounce, dayjs helpers, FileUploader), or is installing frappe-ui into a plain Vue project. For overall project scaffolding, dev proxy, PWA, and deployment of a Frappe SPA, use the frappe-frontend skill; this one is the component-library deep dive.
---

# frappe-ui (component library deep dive)

frappe-ui is the Vue 3 component library behind Frappe's own products (Helpdesk, CRM, Insights, Gameplan, HRMS). It bundles three things:

1. **Components** — buttons, forms, overlays, lists, rich text, file upload.
2. **A Tailwind preset** — Frappe's design language as tokens (type scale, grays, semantic `ink/surface/outline` classes, dark mode).
3. **A data layer** — `createResource` / `createListResource` / `createDocumentResource` wrapping the Frappe REST API.

Docs live at **ui.frappe.io**; source at `github.com/frappe/frappe-ui`. The library moves fast and props change between versions — the installed `node_modules/frappe-ui` source is the final authority, and it's small enough to read.

## How to use this skill

| Task | Reference file |
|---|---|
| Installing into a project, plugin/config setup, Tailwind preset, semantic tokens, dark mode, icons | `references/setup-and-theming.md` |
| Component catalog — forms, overlays, lists, content, feedback — with props and gotchas | `references/components.md` |
| Utilities & composables: `call`, formatters, debounce, FileUploader flow, TextEditor, keyboard shortcuts | `references/utils-and-composables.md` |

For the data layer (`createResource` family, session/auth, realtime), use the frappe-frontend skill's `references/frappe-ui-data.md` — it is not duplicated here.

## Ground rules

- **Verify against the installed version.** Before asserting a prop or slot exists, check `node_modules/frappe-ui/src/components/<Name>.vue` (or the docs site for the version in use). Answers from memory about props are the main source of bugs with this library.
- **Prefer semantic tokens** (`text-ink-gray-8`, `bg-surface-white`, `border-outline-gray-2`) over raw palette classes in new code — they make dark mode free. In older codebases that use `text-gray-700`-style classes, stay consistent instead of mixing.
- **Don't mix design systems.** When frappe-ui lacks a component, copy the composition pattern from a Frappe product repo (Helpdesk, CRM, Insights, Gameplan) rather than pulling in Vuetify/PrimeVue/etc.
- frappe-ui works in any Vue 3 app, but its data layer assumes a Frappe backend. In a non-Frappe app you can still use the components + preset; just skip `resourcesPlugin`/`frappeRequest`.
