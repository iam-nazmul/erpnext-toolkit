# frappe-ui Component Catalog

Import individually: `import { Button, Dialog } from 'frappe-ui'`. Props drift between versions — when precision matters, read `node_modules/frappe-ui/src/components/<Name>.vue` or ui.frappe.io for the installed version. This catalog covers the dependable core and its gotchas.

## Actions

**Button** — `variant`: `solid | subtle | outline | ghost`; `theme`: `gray | blue | green | red`; `size`: `sm | md | lg | xl | 2xl`. Extras: `:loading` (spinner + disabled), `:disabled`, `icon` (icon-only), `icon-left` / `icon-right`, `:route` (renders a router-link), `link` (external href). Icon props accept a feather icon name or a component (Lucide) depending on version.

```vue
<Button variant="solid" :loading="doc.save.loading" icon-left="check" @click="save">Save</Button>
```

**Dropdown** — actions menu:

```vue
<Dropdown :options="[
  { label: 'Edit', icon: 'edit', onClick: edit },
  { group: 'Danger', items: [{ label: 'Delete', icon: 'trash-2', onClick: del, theme: 'red' }] },
]">
  <Button icon="more-horizontal" variant="ghost" />
</Dropdown>
```

Options can be flat or grouped (`{ group, items }`); `condition: () => bool` hides an item.

## Form inputs

**FormControl** — one wrapper for most input types via `type`: `text | number | email | password | date | datetime-local | select | checkbox | textarea | autocomplete`. Common props: `label`, `placeholder`, `description`, `required`, `disabled`. `v-model` throughout; `select` and `autocomplete` take `:options` (strings or `{ label, value }`).

**Autocomplete** — searchable select; for DocType link-field behavior, feed it from a list resource that reloads on search:

```vue
<Autocomplete :options="customers.data" v-model="selected" placeholder="Select customer"
  @update:query="(q) => { customers.filters = { customer_name: ['like', `%${q}%`] }; customers.reload() }" />
```

Product codebases often wrap this as a `Link`/`LinkField` component — copy that pattern for repeated use.

**Select / Checkbox / Textarea / DatePicker / DateTimePicker / DateRangePicker** also exist standalone; `FormControl` covers the common cases with consistent label styling. **Switch** for toggle rows, **Rating** for stars.

## Overlays

**Dialog** — `v-model` for visibility; quick form via `:options`, arbitrary content via slots:

```vue
<Dialog v-model="show" :options="{ title: 'Confirm', message: 'Close this ticket?', size: 'xl',
  actions: [{ label: 'Close Ticket', variant: 'solid', onClick: ({ close }) => closeTicket().then(close) }] }" />

<Dialog v-model="show" :options="{ title: 'New Item' }">
  <template #body-content> <FormControl label="Name" v-model="name" /> </template>
</Dialog>
```

`confirmDialog({ title, message, onConfirm })` (newer versions) for one-off confirms without template plumbing.

**Popover** — anchored floating panel (slots `#target` and `#body-content`). **Tooltip** — `<Tooltip text="…">`. **Toast** — `toast({ title, text, icon, position })` or `toast.success(...)` / `toast.error(...)` per version; mount `<Toasts />` once at app root if the version requires it.

## Lists & navigation

**ListView** — the data-table workhorse: `:columns="[{ label, key, width }]"`, `:rows`, `row-key`, `:options="{ selectable, showTooltip, resizeColumn, getRowRoute | onRowClick }"`. Compose with `ListHeader`, `ListRows`, `ListRow`, `ListSelectBanner` for custom layouts; column `prefix`/`suffix` slots render badges/avatars in cells. For server-driven pagination pair with `createListResource` + `hasNextPage`/`next()`.

**Tabs** — `v-model="tabIndex"` + `:tabs="[{ label, icon }]"`, content via `#tab-panel` slot. **Breadcrumbs** — `:items="[{ label, route }]"`. **Badge** — `variant`/`theme` like Button. **Avatar** — `:image`, `:label` (initials fallback), `size`. **Divider**, **Card** (simple titled container; many products just use divs + tokens).

## Content & feedback

**TextEditor** — TipTap rich text, used for comments/descriptions across Frappe products: `:content`, `@change`, `:mentions`, `editor-class` for styling, `:bubbleMenu`/`:fixedMenu` toolbars. HTML in/out — sanitize server-side.

**FileUploader** — wraps Frappe's `upload_file` (see `utils-and-composables.md`).

**ErrorMessage** — `:message="resource.error"`; renders nothing when empty, so it can sit permanently under a form. **Spinner** / **LoadingIndicator**, **Progress**, **FeatherIcon**.

## When a component is missing

Kanban boards, filter builders, comment threads, sidebar layouts, command palettes — frappe-ui doesn't ship these, but every Frappe product has them. Copy from **Helpdesk, CRM, Gameplan, Insights** repos (they all use this exact stack) instead of adding a second UI library.
