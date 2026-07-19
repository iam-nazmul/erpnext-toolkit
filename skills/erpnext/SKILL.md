---
name: erpnext
description: Work with ERPNext and the Frappe Framework — integrate via the REST API, build custom apps and DocTypes, write server/client scripts, create reports, automate business workflows (sales, purchasing, stock, accounting, HR), and troubleshoot bench/site issues. Use this skill whenever the user mentions ERPNext, Frappe, bench, DocTypes, or wants to read/write ERP data (customers, sales orders, invoices, items, stock, employees), customize forms, build integrations against an ERPNext instance, or generate ERPNext reports — even if they don't say "ERPNext" explicitly but describe a Frappe-based system.
---

# ERPNext / Frappe Framework

ERPNext is an open-source ERP built on the **Frappe Framework** (Python + MariaDB + Redis). Everything in the system — invoices, customers, settings, even reports themselves — is a **DocType**. Understanding DocTypes unlocks everything else.

## How to use this skill

1. Read the **Core concepts** below (short — always worth it).
2. Then open the reference file matching the task:

| Task | Reference file |
|---|---|
| Call ERPNext from outside (integrations, scripts, other apps) | `references/rest-api.md` |
| Build a custom app, new DocTypes, hooks, scheduled jobs | `references/frappe-development.md` |
| Customize without code deployment (server/client scripts, custom fields) | `references/low-code-customization.md` |
| Build Query Reports or Script Reports | `references/reports.md` |
| Automate business flows (sales cycle, purchasing, stock, payments) | `references/business-workflows.md` |
| Custom portal / PWA / Vue frontend on top of Frappe | the companion **frappe-frontend** skill (Vue 3 + frappe-ui + Tailwind), if installed |
| frappe-ui component library specifics (components, theming, composables) | the companion **frappe-ui** skill, if installed |
| BI dashboards, queries, charts in Frappe Insights | the companion **frappe-insights** skill, if installed |
| Code review / security audit of a Frappe app | the companion **frappe-code-review** skill, if installed |

Read only what the task needs. For mixed tasks (e.g., "a script report that a webhook triggers"), read both relevant files.

## Core concepts

**DocType** — a document type: both the schema (fields, permissions, naming) and the table (`tabSales Invoice` in MariaDB). Documents are rows. Every DocType is itself stored as a document of DocType "DocType".

**Naming** — every document has a unique `name` (the primary key). Naming can be by field, by naming series (`SINV-.YYYY.-.#####`), autoincrement, or prompt. When calling APIs you always address documents as `(doctype, name)`.

**docstatus** — transactional documents have a lifecycle: `0` = Draft, `1` = Submitted, `2` = Cancelled. **Submitted documents are immutable** except for fields explicitly marked "Allow on Submit". You cannot edit or delete a submitted doc — you cancel it (and usually amend, which creates `NAME-1`). This is the single most common source of confusion; check docstatus before attempting writes.

**Child tables** — a DocType field of type "Table" points to a child DocType (e.g., Sales Invoice → Sales Invoice Item). Child rows live in their own table with `parent`, `parenttype`, `parentfield` columns. Via API, child rows are sent/received as nested lists in the parent doc.

**Link fields** — foreign keys to another DocType's `name` (e.g., `customer` on Sales Invoice links to Customer). Values must be an existing document name, not a label.

**Sites and apps** — one bench (installation) hosts multiple **sites** (databases); each site has **apps** installed (frappe, erpnext, custom apps). Most bench commands need `--site sitename` or `--site all`.

**Permissions** — role-based, per DocType, per permission level, optionally row-level via User Permissions. API calls run with the permissions of the authenticated user; a 403 usually means missing role permission, not a wrong endpoint.

**Server vs client** — Python runs on the server (controllers, server scripts, whitelisted methods); JavaScript runs in the browser (client scripts, form events). The two communicate through `frappe.call` → whitelisted Python methods.

## Version notes

Assume **ERPNext v15 / Frappe v15** unless the user says otherwise; most guidance also applies to v13/v14. Notable differences worth confirming with the user:
- v15 ships a `/api/v2` REST interface; `/api/resource` (v1) still works everywhere and is used in `rest-api.md`.
- HR and Payroll moved out of ERPNext core into the separate **HRMS** app from v14 onward.
- Workspaces replaced the old Desk modules page in v13+.

## Ground rules when helping users

- **Never mutate submitted documents directly in SQL.** Ledger tables (GL Entry, Stock Ledger Entry) are derived — editing them corrupts accounting. Fix problems through cancel/amend or official repair tools.
- Prefer the official surface in this order: REST API / `frappe.get_doc` ORM → server scripts → custom app code → raw SQL (read-only analysis only).
- ERPNext validation lives in controllers; bypassing it (`db_insert`, direct SQL) skips GL/stock postings. Only bypass for migrations with eyes open and say so explicitly.
- When the user reports an error, ask for (or check) the full traceback from the browser console or `bench --site <site> console` / error log — Frappe tracebacks name the exact controller line.
- If working against a live instance, confirm whether it is production before performing writes, bulk updates, or cancellations.
