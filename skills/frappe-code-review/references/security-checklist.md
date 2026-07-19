# Frappe Security Checklist

Work through these per entry point. Each item: what to look for, why it's dangerous, the safe form.

## 1. Whitelisting & authentication

- **Every `@frappe.whitelist()` is a public-once-logged-in endpoint.** Enumerate them (`grep -rn "@frappe.whitelist" app/`). For each, ask: what does it read/write, and does it check that *this* user is allowed to?
- **`allow_guest=True`** = reachable with no login. Justified only for genuinely public actions (contact form, public data). It must not touch anything user-scoped or perform privileged writes. Treat each one as internet-facing.
- **`methods=["POST"]`** should be set on any whitelisted method that mutates state — otherwise it's callable via GET, which makes CSRF and accidental crawling easier.
- Methods called only internally should **not** be whitelisted. Whitelisting "to make the JS call work" widens the surface; if the frontend needs it, it still needs its own permission checks.
- Guest-callable methods returning lists must not leak other users' rows — verify filters are server-enforced, not passed by the client.

## 2. Authorization / permission bypass (the big one)

- **User-supplied `doctype` / `name` / `filters`** flowing into `frappe.get_doc`, `get_all`, `get_list`, `db.get_value`, `delete_doc` → classic **IDOR**. `frappe.get_list` respects permissions; `frappe.get_all` and `db.*` do **not**. A whitelisted method doing `frappe.get_doc(dt, dn)` on caller-controlled `dn` must call `doc.has_permission("read")` / `frappe.has_permission(dt, "write", doc)` before returning or mutating.
- **`ignore_permissions=True`** — grep for it. Each occurrence must be justified by running in trusted/system context, never gated only by the fact that it's "internal".
- **`frappe.only_for("System Manager")`** or explicit role checks should guard admin-ish methods; absence on a privileged method is a finding.
- **Mass assignment**: `doc.update(frappe.form_dict)` / `doc.update(kwargs)` lets a caller set *any* field, including `owner`, `docstatus`, workflow_state, or amount fields. Whitelist the settable fields explicitly.
- **Row-level**: even with DocType permission, User Permissions may be expected to scope rows (e.g. a user's own Company). Bypassing the ORM skips these.

## 3. SQL injection

- **`frappe.db.sql`** does not escape interpolated values. Any f-string / `%`-format / `.format()` / string-concat into the query with user input = injection.
  ```python
  # VULNERABLE
  frappe.db.sql(f"select * from `tabItem` where item_group = '{group}'")
  # SAFE — parameterized
  frappe.db.sql("select * from `tabItem` where item_group = %(g)s", {"g": group})
  ```
- **`order_by` / `fields` / column names** can't be parameterized — if user-controlled, validate against an allowlist; injection here is common and often missed.
- **`filters` strings** passed raw to query builder: prefer dict/list filters. Report Builder / dynamic query features that accept raw conditions are a known risk area.
- Prefer the ORM (`frappe.get_all`, Query Builder / `frappe.qb`) over `db.sql`; flag raw SQL that has no reason to be raw.

## 4. Secrets & sensitive data

- No hardcoded API keys/passwords/tokens in code or committed `site_config.json`. Secrets belong in `site_config.json` (untracked) or env, read via `frappe.conf`.
- Password/secret DocType fields should use fieldtype `Password` (encrypted at rest) and be read via `get_password()`, not `db.get_value` (which returns the decrypted value and can leak into logs/queries).
- **Error messages / `frappe.throw`** must not echo internal data or full tracebacks to guests. `frappe.log_error` for internals; user-facing messages stay generic.
- Whitelisted methods returning a whole `doc` may over-expose fields (hidden/permlevel fields, other users' PII). Return an explicit projection.

## 5. Server-side request & command execution

- **SSRF**: `requests.get(user_url)`, webhooks, "fetch from URL" features — validate/allowlist the destination; block internal IPs/metadata endpoints.
- **Command/eval**: `os.system`, `subprocess` with user input, `eval`/`exec`, `frappe.safe_eval` with attacker-controlled expressions. Server Scripts run sandboxed-ish but review any dynamic execution.
- **Path traversal**: user-supplied filenames/paths into file reads/writes (`../`), especially in import/export and attachment features.

## 6. File uploads & attachments

- Validate content type and extension server-side; don't trust the client. Frappe stores public files under `/files` (world-readable via URL) and private under `/private/files` — sensitive attachments must be `is_private`.
- Guard against unrestricted upload → stored XSS (SVG/HTML) served from the site origin.

## 7. CSRF & sessions

- Frappe enforces CSRF tokens on form-based POSTs by default; a custom endpoint disabling it (`ignore_csrf`) or a frontend not sending `X-Frappe-CSRF-Token` is a finding (see `frontend-review.md`).
- Long-lived API keys/secrets: confirm they're scoped and rotatable; a shared admin key in a client app is a finding.

## Severity guidance

- **Critical**: unauthenticated write/RCE, permission bypass exposing financial/PII data, SQL injection reachable by a low-priv user.
- **High**: authenticated IDOR, `ignore_permissions` reachable from user input, secret exposure.
- **Medium**: missing `methods=["POST"]`, over-broad data return, SSRF with limited impact.
- **Low/Info**: defense-in-depth, style, hardening suggestions.
