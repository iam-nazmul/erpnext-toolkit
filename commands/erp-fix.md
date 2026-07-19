---
description: Diagnose and fix ERPNext/Frappe errors — tracebacks, bench issues, permission errors, stuck documents
argument-hint: [paste the error or describe the problem]
---

The user has an ERPNext/Frappe problem: $ARGUMENTS

Follow these steps:

1. Read the `erpnext` skill Core concepts. Open the reference file matching the problem area.
2. If no traceback was provided, ask for it — from the browser console, the Error Log DocType, or `bench --site <site> console`. Frappe tracebacks name the exact controller line; don't guess without one.
3. Common diagnoses to check first:
   - **403 / PermissionError** → missing role permission or User Permission row-level restriction, not a wrong endpoint.
   - **Cannot edit document** → it's submitted (`docstatus = 1`); the fix is cancel/amend, never direct edits.
   - **TimestampMismatchError** → document was modified elsewhere; reload and retry.
   - **LinkValidationError** → a Link field contains a label or a non-existent document name.
   - **bench errors** → check whether `--site` was specified, whether migrations are pending (`bench --site <site> migrate`), and supervisor/redis status.
4. Safety rules (non-negotiable):
   - Never fix accounting/stock problems by editing GL Entry or Stock Ledger Entry via SQL — these are derived tables; editing them corrupts the books. Use cancel/amend or official repair tools.
   - Before any write/bulk fix on a live instance, confirm whether it is production.
5. Give the fix with exact commands or steps, and explain the root cause in one or two sentences so it doesn't recur.
