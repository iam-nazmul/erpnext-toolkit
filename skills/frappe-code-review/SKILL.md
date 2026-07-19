---
name: frappe-code-review
description: Review, analyze, and security-test a Frappe/ERPNext codebase — custom apps, DocType controllers, whitelisted API methods, server/client scripts, hooks, and Vue/frappe-ui frontends. Use this skill whenever the user wants a code review, architecture/quality analysis, or security assessment of Frappe/ERPNext code; asks "is this safe/correct", "review my app", "audit permissions", "find vulnerabilities", or mentions concerns like SQL injection, permission bypass, unsafe `frappe.db.sql`, missing `@frappe.whitelist` guards, `ignore_permissions`, or exposed API methods. Applies to authorized review of code you have access to — not attacking live third-party systems. For generic (non-Frappe) diff review use the built-in /code-review and /security-review commands; this skill layers Frappe-specific rules on top.
---

# Frappe / ERPNext Code Review & Security Testing

Review Frappe apps the way Frappe breaks: the framework is permissive by default, so most real bugs are **missing guards** (permission checks, whitelist arguments, validation) rather than exotic exploits. This skill gives a Frappe-aware checklist and workflow layered on top of the generic `/code-review` and `/security-review` commands.

## How to use this skill

1. **Scope it.** Identify what you're reviewing: a diff/PR, a whole custom app, or a running site. Confirm authorization and whether the target is production before running anything that writes or probes.
2. Read **Review workflow** below, then open the reference for the dimension you need.
3. Report findings ranked by severity with `file:line`, the concrete failure scenario, and the fix — same discipline as `/code-review`.

| Task | Reference file |
|---|---|
| Security review: permissions, whitelisting, SQL injection, secrets, SSRF, uploads | `references/security-checklist.md` |
| Code quality & correctness: controllers, transactions, ORM misuse, performance, N+1 | `references/code-quality.md` |
| Frontend review: frappe-ui/Vue SPA, XSS, CSRF, exposed data, client-trust bugs | `references/frontend-review.md` |
| How to actually run the analysis: grep patterns, tooling, dynamic testing on a bench | `references/how-to-test.md` |

## Review workflow

1. **Map the attack surface.** List every entry point: `@frappe.whitelist()` methods, `hooks.py` (`override_whitelisted_methods`, `doc_events`, scheduled jobs), server scripts, `www/` pages and portal routes, REST-exposed DocTypes. Untrusted input arrives at all of these.
2. **Follow the data.** For each entry point trace user input → DB. The dangerous transitions are: user string → `frappe.db.sql`; user-supplied `doctype`/`name` → `get_doc`/`delete` without permission check; user dict → `doc.update()` / `db_set` (mass assignment).
3. **Check the guards at each transition** using the security checklist.
4. **Then assess quality/correctness** (transactions, docstatus handling, performance) using the code-quality reference.
5. **Verify dynamically where feasible** — a claim of "SQL injection here" should be demonstrated on a dev bench, not asserted. See `how-to-test.md`.

## The five Frappe-specific red flags (memorize these)

1. **`frappe.db.sql(f"... {user_input} ...")`** — string-formatted SQL. Frappe's `db.sql` does *not* auto-escape; this is the #1 injection vector. Must use parameterized `values=` / `%(name)s`.
2. **`@frappe.whitelist(allow_guest=True)`** — the method is reachable by anyone, unauthenticated. Every one is a public endpoint; scrutinize what it reads/writes.
3. **`ignore_permissions=True`** (on `get_doc`, `.save`, `.insert`, `frappe.get_all(... ignore_permissions=True)`) — bypasses the permission system. Legitimate in system context; a bug when reachable from user input.
4. **A whitelisted method that takes `doctype`/`name`/`filters` from the caller** and passes them to the ORM without `frappe.has_permission` — lets a user read/write documents they shouldn't (IDOR).
5. **`frappe.db.set_value` / `db.sql UPDATE` on ledger or submitted docs** — corrupts accounting/stock and skips validation; almost never correct outside a guarded migration.

## Ground rules

- **Authorized targets only.** Review code the user owns or is engaged to assess; dynamic testing goes against a dev/staging bench, never someone else's production. Decline requests aimed at attacking third-party systems.
- Prefer **demonstrated** findings over speculation; mark anything unverified as such and say how to confirm it.
- Frappe's own source is the reference for "is this the safe API" — when unsure whether a call escapes input or checks permissions, read the framework method rather than guessing.
- Report defensively: severity, exact location, exploit/failure scenario, and the minimal fix. Don't bury a critical permission bypass under style nits.
