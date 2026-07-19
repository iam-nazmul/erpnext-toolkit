# frappe-ui Utilities & Composables

Beyond components and resources, frappe-ui exports small utilities that products lean on constantly. Exact export names vary a little by version — `node_modules/frappe-ui/src/index.js` (or `src/utils/`) lists what the installed version actually ships.

## call — one-off API requests

```js
import { call } from 'frappe-ui'
const result = await call('my_app.api.do_thing', { x: 1 })  // returns `message`, throws on error
```

Use for imperative actions (button handlers) where a reactive resource is overkill. It goes through `frappeRequest`, so CSRF and error parsing are handled.

## File uploads

`FileUploader` component wraps Frappe's `upload_file` endpoint with chunking and progress:

```vue
<FileUploader
  :fileTypes="['image/*']"
  :uploadArgs="{ doctype: 'HD Ticket', docname: ticket.name, private: true }"
  @success="(file) => attachments.push(file)"  <!-- file.file_url is the uploaded path -->
>
  <template #default="{ openFileSelector, uploading, progress }">
    <Button @click="openFileSelector" :loading="uploading">
      {{ uploading ? `Uploading ${progress}%` : 'Attach file' }}
    </Button>
  </template>
</FileUploader>
```

There's also a `FileUploadHandler` class for programmatic uploads (drag-drop zones, paste-to-upload). Private files require the session to have read access to the linked doc.

## Date & time helpers

frappe-ui bundles dayjs preconfigured with relative-time and timezone plugins:

```js
import { dayjs } from 'frappe-ui'
dayjs(doc.modified).fromNow()          // "4 hours ago"
dayjs(doc.date).format('D MMM YYYY')
```

Newer versions add format helpers (`formatDate`, `timeAgo`-style utilities) — check exports before writing your own. Frappe server datetimes are strings in the site's timezone; be deliberate when converting.

## Formatting & misc utilities

Commonly exported (verify per version):

- `debounce(fn, ms)` — the standard companion to search inputs feeding a list resource.
- Number/currency/bytes formatters (`formatNumber`, `fileSizeText`-style helpers) used by product UIs.
- Text helpers used by TextEditor integrations (HTML→text, mention extraction) live under `src/utils`.

If a helper isn't exported, importing from `frappe-ui/src/utils/...` works (source is shipped) but pins you to internals — copy the function into your project instead when it's tiny.

## Keyboard shortcuts & focus

Recent versions expose composables like `useShortcuts`/`onKeyDown`-style helpers and products use plain `@keydown` + a command-palette pattern (see Helpdesk/CRM source). `onOutsideClickDirective` / `v-on-outside-click` is exported for closing custom popovers:

```js
import { onOutsideClickDirective } from 'frappe-ui'
app.directive('on-outside-click', onOutsideClickDirective)
```

## Composition patterns worth copying

- **Search input → debounced resource reload**: `watch(query, debounce(() => { list.filters = ...; list.reload() }, 300))`.
- **Optimistic toggle**: `resource.setData(...)` immediately, then `setValue.submit(...)`; reload on error.
- **`computed` view-models over `resource.data`** rather than transforming in templates; keep `transform()` on the resource for shaping server data once.
- **Global stores**: products use plain reactive modules (like the session pattern in the frappe-frontend skill) or Pinia — frappe-ui doesn't impose one.
