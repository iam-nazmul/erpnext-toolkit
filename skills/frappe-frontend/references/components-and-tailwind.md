# frappe-ui Components & TailwindCSS

## Tailwind setup (the frappe-ui way)

frappe-ui ships a Tailwind **preset** that defines Frappe's design language. Use it instead of stock Tailwind defaults:

```js
// tailwind.config.js
module.exports = {
  presets: [require('frappe-ui/src/tailwind/preset')],
  content: [
    './index.html',
    './src/**/*.{vue,js,ts}',
    './node_modules/frappe-ui/src/components/**/*.{vue,js}',  // required — or component styles get purged
  ],
}
```

(The preset path has moved between versions — older releases used `frappe-ui/src/utils/tailwind.config`; check `node_modules/frappe-ui` for the current one. Scaffolds come pre-wired.)

What the preset changes, and the conventions to follow:

- **Denser type scale** than stock Tailwind — `text-base` is UI-sized (~14px), with steps like `text-sm`, `text-lg`, `text-xl` recalibrated for app UIs, plus paragraph sizes (`text-p-sm`, `text-p-base`, …) in newer versions. Don't assume stock Tailwind pixel values.
- **Extended gray palette** tuned to Frappe products (gray-50…gray-900); grays are the backbone of the UI, brand color is used sparingly.
- **Semantic tokens** (newer frappe-ui): `text-ink-gray-8`, `text-ink-gray-5`, `bg-surface-white`, `bg-surface-gray-2`, `border-outline-gray-2`, etc. Prefer these over raw palette classes in new code — they make dark mode and theming automatic. Older codebases use raw `text-gray-700`-style classes; stay consistent with whatever the project uses.
- Standard Tailwind workflow otherwise: utility classes in templates, `@apply` sparingly, no separate CSS files unless necessary.

Rules of thumb for the Frappe look: white/gray surfaces, `rounded` / `rounded-lg` corners, subtle `shadow-sm`, 1px `border-outline-gray-*` borders, small type, generous whitespace, medium-weight labels.

## Components

Import individually: `import { Button, Dialog, FormControl } from 'frappe-ui'`. Props evolve between versions — verify against the installed version's source or the docs site (ui.frappe.io) when precision matters. The dependable core:

**Button**

```vue
<Button variant="solid" theme="gray" size="sm" :loading="res.loading" @click="go">
  Save
</Button>
```

Variants: `solid | subtle | outline | ghost`; themes: `gray | blue | green | red`. `icon`/`icon-left`/`icon-right` props take a (feather) icon name; `:route` makes it a router-link.

**FormControl** — the one input to rule them all:

```vue
<FormControl type="text"     label="Subject" v-model="doc.subject" placeholder="…" />
<FormControl type="select"   label="Status"  v-model="doc.status" :options="['Open','Closed']" />
<FormControl type="checkbox" label="Urgent"  v-model="doc.urgent" />
<FormControl type="textarea" label="Notes"   v-model="doc.notes" />
<FormControl type="autocomplete" label="Customer" v-model="customer" :options="customerOptions" />
```

For Link-field behavior (search a DocType), use the `Autocomplete` component (or `LinkField`-style wrappers in product codebases) fed by a `createListResource` that reloads on search text.

**Dialog**

```vue
<Dialog v-model="showDialog" :options="{
  title: 'Confirm',
  message: 'Close this ticket?',
  actions: [{ label: 'Close Ticket', variant: 'solid', onClick: closeTicket }],
}" />
```

Or use the `#body-content` slot for arbitrary content. There's also a promise-based `confirmDialog` utility in recent versions.

**Lists**: `ListView` (with `ListHeader`, `ListRow`, selection, grouping) for data tables; or roll your own with `v-for` for simple cases.

**Feedback & overlays**: `toast()` / `Toasts` for notifications, `Tooltip`, `Popover`, `Dropdown` (`:options="[{label, onClick}]"`), `Badge`, `Avatar`, `Spinner`/`LoadingIndicator`, `ErrorMessage :message="res.error"`.

**Content**: `TextEditor` (TipTap-based rich text — used for comments/descriptions across Frappe products), `FileUploader` (wraps Frappe's upload_file; slot props give `progress`, `uploading`, and the resulting file URL), `Tabs`, `Breadcrumbs`, `FeatherIcon name="x"`.

## Composition example (a filtered list page)

```vue
<script setup>
import { ref, watch } from 'vue'
import { Button, FormControl, ListView, Badge } from 'frappe-ui'
import { createListResource } from 'frappe-ui'

const status = ref('Open')
const tickets = createListResource({
  doctype: 'HD Ticket',
  fields: ['name', 'subject', 'status'],
  filters: { status: status.value },
  auto: true,
})
watch(status, (v) => { tickets.filters = { status: v }; tickets.reload() })
</script>

<template>
  <div class="p-4 space-y-4">
    <div class="flex items-center justify-between">
      <h1 class="text-xl font-semibold text-ink-gray-8">Tickets</h1>
      <FormControl type="select" v-model="status" :options="['Open', 'Closed']" />
    </div>
    <ListView :columns="[{label:'Subject', key:'subject'},{label:'Status', key:'status'}]"
              :rows="tickets.data || []" row-key="name" />
  </div>
</template>
```

## When frappe-ui doesn't have it

Check how Frappe's open-source products solved it first — **Helpdesk, CRM, Insights, Gameplan, HRMS** repos are the best pattern library for this stack (kanban boards, filters UI, comment threads, mobile layouts). Copying their component composition beats bringing in a second UI library; mixing frappe-ui with another design system usually looks wrong.
