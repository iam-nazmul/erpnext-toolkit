---
description: Build an ERPNext Query Report or Script Report with filters and formatted columns
argument-hint: [describe the report, e.g. "monthly sales by territory with year filter"]
---

The user wants an ERPNext report: $ARGUMENTS

Follow these steps:

1. Read the `erpnext` skill — Core concepts plus `references/reports.md`.
2. Choose the report type:
   - **Query Report** (SQL) for straightforward tabular data
   - **Script Report** (Python) when you need computed columns, multiple queries, or chart data
3. Verify table and column names before writing SQL — tables are named `tab<DocType>` (e.g. `tabSales Invoice`). Use the MCP tools or DocType field lists to confirm field names; never invent them.
4. Standard rules:
   - Filter transactional data by `docstatus = 1` unless the user wants drafts.
   - Define filters with proper fieldtypes (Link, Date Range, Select) so the report UI renders them.
   - Format columns with `fieldtype` (Currency, Date, Link) so amounts and links display correctly.
5. Deliver the complete report: SQL or Python code, filter definitions, column definitions, and step-by-step instructions to create it via the Report DocType (Report Type: "Query Report" or "Script Report").

If the data need is one-off rather than a reusable report, suggest just querying via `/erp-query` instead.
