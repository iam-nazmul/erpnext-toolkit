---
description: Design and create a new DocType (schema, fields, permissions, controller) for a custom Frappe app
argument-hint: [describe the DocType, e.g. "Vehicle Maintenance Log with odometer, cost, workshop"]
---

The user wants to create a new DocType: $ARGUMENTS

Follow these steps:

1. Read the `erpnext` skill — Core concepts plus `references/frappe-development.md`.
2. Clarify anything essential that's missing (only if truly needed): which app it belongs to, whether it's submittable (has docstatus lifecycle), naming scheme (naming series vs field vs autoincrement), and who should have access (roles).
3. Design the schema:
   - Choose correct fieldtypes (Data, Link, Select, Currency, Table for child tables, etc.).
   - Link fields must point to existing DocTypes.
   - If line items are needed, design the child DocType too (`istable: 1`).
4. Produce the deliverables:
   - The DocType JSON definition(s)
   - A Python controller with validation hooks (`validate`, `on_submit`, etc.) if business logic is needed
   - Permission rules per role
5. Explain how to install it: place files in the custom app and run `bench --site <site> migrate`, or create it via the DocType form in the UI for quick prototyping.

Prefer creating DocTypes inside a custom app (not directly in erpnext/frappe apps). If the user only needs extra fields on an existing DocType, suggest Custom Fields instead — see `references/low-code-customization.md`.
