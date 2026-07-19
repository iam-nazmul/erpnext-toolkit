# frappe-ui Data Layer (resources, auth, realtime)

frappe-ui wraps Frappe's REST API in reactive "resources". Setup once in `main.js`:

```js
import { createApp } from 'vue'
import { setConfig, frappeRequest, resourcesPlugin } from 'frappe-ui'
import App from './App.vue'

setConfig('resourceFetcher', frappeRequest)   // teaches resources to call Frappe (CSRF, errors, /api prefix)
const app = createApp(App)
app.use(resourcesPlugin)
app.mount('#app')
```

## createResource — one API method

```js
import { createResource } from 'frappe-ui'

const stats = createResource({
  url: 'my_app.api.get_dashboard_stats',   // any whitelisted method; 'login' etc. also work
  params: { period: 'month' },
  auto: true,                              // fetch immediately
  cache: 'dashboard-stats',                // dedupe across components
  transform(data) { return data.rows },
  onSuccess(data) {},
  onError(err) {},
})
```

Reactive fields: `stats.data`, `stats.loading`, `stats.error`, `stats.fetched`. Methods: `stats.fetch()`, `stats.reload()`, `stats.submit(params)` (fetch with new params), `stats.setData(fn|value)` (optimistic updates).

In templates: `<div v-if="stats.loading">…</div>` then render `stats.data`.

## createListResource — a DocType list with pagination & mutations

```js
const tickets = createListResource({
  doctype: 'HD Ticket',
  fields: ['name', 'subject', 'status', 'modified'],
  filters: { status: 'Open' },
  orderBy: 'modified desc',
  pageLength: 20,
  auto: true,
  cache: ['tickets', 'open'],
})
```

- Read: `tickets.data` (array), `tickets.loading`, `tickets.hasNextPage`, `tickets.next()` (infinite scroll), `tickets.reload()`.
- Change query: `tickets.update({ filters: {...} })` then `tickets.reload()` — or just mutate `tickets.filters` and reload.
- Mutations (each is itself a resource with `.loading`/`.error`):

```js
tickets.insert.submit({ subject: 'New', status: 'Open' })
tickets.setValue.submit({ name: 'TCK-0001', status: 'Closed' })
tickets.delete.submit('TCK-0001')
```

List data refreshes automatically after successful mutations.

## createDocumentResource — one document + its methods

```js
const ticket = createDocumentResource({
  doctype: 'HD Ticket',
  name: props.ticketId,
  auto: true,
  whitelistedMethods: { markSeen: 'mark_seen' },   // controller methods exposed via @frappe.whitelist()
})
```

- `ticket.doc` — the reactive document (children included).
- `ticket.setValue.submit({ status: 'Replied' })` — persists a partial update.
- `ticket.save.submit()` — save the (locally edited) doc.
- `ticket.delete.submit()`, `ticket.reload()`, `ticket.markSeen.submit()`.

Prefer `setValue` for field edits (server-validated, minimal payload) over mutating `doc` and saving.

## Direct calls

For one-off calls without reactivity: `import { call } from 'frappe-ui'` → `await call('my_app.api.do_thing', { x: 1 })`. Returns the `message` payload, throws on error.

## Session / auth pattern

The starter's `src/data/session.js` is the canonical pattern:

```js
import { createResource } from 'frappe-ui'
import { computed, reactive } from 'vue'

export function sessionUser() {
  const cookies = new URLSearchParams(document.cookie.split('; ').join('&'))
  let user = cookies.get('user_id')
  return user && user !== 'Guest' ? user : null
}

export const session = reactive({
  user: sessionUser(),
  isLoggedIn: computed(() => !!session.user),
  login: createResource({
    url: 'login',
    onSuccess() { session.user = sessionUser(); /* redirect */ },
  }),
  logout: createResource({
    url: 'logout',
    onSuccess() { session.user = null; window.location.href = '/login' },
  }),
})

// usage: session.login.submit({ usr, pwd })
```

Router guard: redirect to your login route when `!session.isLoggedIn` on protected routes. Remember: this is UX only — real authorization is server-side permissions.

## Realtime (socket.io)

```js
import { initSocket } from 'frappe-ui'   // or the socket helper in your scaffold's src/socket.js
const socket = initSocket()
socket.on('doc_update', () => tickets.reload())
```

Server side: `frappe.publish_realtime('event_name', data, user=...)`. In dev, ensure the socketio port (default 9000) is proxied/reachable; in production nginx handles `/socket.io`. Realtime is an optimization — always make reload/polling work first.

## Error handling conventions

`frappeRequest` throws structured errors; resources expose them at `resource.error`. Show `resource.error.messages?.join(', ') || resource.error.message` in an `<ErrorMessage :message="..." />` component. Validation errors raised with `frappe.throw()` server-side arrive as readable messages — lean on server validation instead of duplicating rules client-side.
