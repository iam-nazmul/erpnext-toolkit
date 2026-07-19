---
description: Fetch and analyze data from the connected ERPNext instance (documents, lists, reports)
argument-hint: [what data you want, e.g. "unpaid sales invoices this month"]
---

The user wants to query data from their ERPNext instance: $ARGUMENTS

Follow these steps:

1. Read the `erpnext` skill (Core concepts + `references/rest-api.md`) if you haven't already in this session.
2. Identify the relevant DocType(s). If unsure, use the MCP tools to list DocTypes or fetch DocType fields first — never guess field names.
3. Fetch the data using the ERPNext MCP tools (preferred) or the REST API (`/api/resource/<DocType>` with `filters`, `fields`, `limit_page_length`).
4. Remember:
   - `docstatus`: 0 = Draft, 1 = Submitted, 2 = Cancelled. For financial/stock queries, usually filter `docstatus = 1`.
   - Link fields hold document `name`s, not labels.
   - Child table rows (e.g. invoice items) come nested inside the parent document.
5. Present the result concisely: a short summary first, then a table if there are multiple rows. Offer to export to a file (CSV/XLSX) if the dataset is large.

Never write or modify data with this command — it is read-only. If the user's request implies a write, confirm explicitly before doing anything, and check whether the instance is production.
