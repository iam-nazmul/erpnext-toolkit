# Frappe Frontend Review (frappe-ui / Vue SPA & Desk client scripts)

The frontend is untrusted — every check it does is UX, not security. Review it for (a) client bugs and (b) places where the app *relies* on the client for safety, which is really a backend finding.

## The cardinal rule

**Anything the client enforces, the server must re-enforce.** A hidden button, a disabled field, a client-side role check, a filtered list — all bypassable via the API/console. When you see client-side authorization, the finding is "is the corresponding server check present?" (trace it to the whitelisted method / DocType permission). If not → permission bypass, report against the backend.

## XSS

- **`v-html`** with any server/user-derived string = stored/reflected XSS. Frappe products render user content (comments, descriptions) through `TextEditor`/sanitized HTML — raw `v-html` of a field value is a finding unless provably sanitized server-side.
- **Desk client scripts / custom HTML fields**: `$(...).html(userValue)`, `frappe.msgprint(userValue)` with HTML, custom HTML fieldtypes echoing data. Use text binding or sanitize.
- **`innerHTML`, `dangerouslySetInnerHTML`-equivalents, `eval`** anywhere in `src/`.
- Markdown/rich-text rendered without sanitization; SVG/HTML file attachments rendered inline (served from site origin → runs in session).

## CSRF & requests

- Production SPA served from the site must send the CSRF token. `frappe-ui`'s `frappeRequest` handles it when `window.csrf_token` is set by the served `index.html` — verify that wiring exists (POSTs failing only in prod with `CSRFTokenError` is the symptom). A custom fetch that omits `X-Frappe-CSRF-Token` on mutations is a finding.
- No API keys/secrets embedded in the bundle — anything in `src/` ships to the browser. Grep the build for tokens.

## Data exposure

- **Over-fetching**: `createListResource`/`createResource` pulling fields the user shouldn't see, then hiding them in the UI — the data already reached the browser. The fix is server-side field permissions / a projection method, not a `v-if`.
- **Client-side filtering of sensitive rows** (`rows.filter(...)` after fetching all) — the unfiltered set is in memory and on the wire.
- Debug logging of documents/PII to the console left in production.

## Client correctness (non-security)

- **Optimistic updates** (`resource.setData`) not rolled back on error → UI shows success that didn't persist.
- **Missing loading/error states**: `ErrorMessage`/`resource.error` unhandled; silent failures.
- **Stale resources**: not calling `reload()` after a mutation done outside the resource, or relying on realtime that isn't guaranteed (see the data-layer reference — reload/polling must work without sockets).
- **Reactivity bugs**: mutating `resource.filters` without `reload()`; destructuring reactive resources (loses reactivity); watchers with wrong deps.
- **Router guards** doing auth checks that flash protected content before redirect; secrets in route params/query.

## frappe-ui-specific checks

- Props/version drift causing silent no-ops (e.g. a renamed prop) — verify against installed `node_modules/frappe-ui` per the frappe-ui skill.
- `FileUploader` with `private: false` for sensitive files → publicly reachable URL.
- `Dialog`/form flows that submit on the client without disabling during in-flight requests (double-submit → duplicate docs; pair with server-side idempotency check).

## How to report

Split findings: client bugs (UX/correctness) vs client-trust bugs that are actually server holes. For the latter, name the backend location that needs the check, since fixing the frontend alone doesn't close the hole.
