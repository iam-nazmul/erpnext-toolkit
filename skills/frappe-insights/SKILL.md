---
name: frappe-insights
description: Set up and use Frappe Insights — the open-source BI/analytics app for Frappe/ERPNext — for data sources, query building (visual or SQL), charts, and dashboards. Use this skill whenever the user mentions Insights, wants BI/analytics/dashboards over ERPNext data, needs to connect an external MariaDB/PostgreSQL database or upload CSVs for analysis, asks about the Insights query builder, calculated columns/expressions, joins across DocTypes, sharing or embedding dashboards, or Insights permissions/teams. Also use when deciding between Insights and built-in ERPNext reports (Query/Script Reports, Desk dashboards) — Insights is for exploratory/self-serve analytics; see the erpnext skill's reports.md for the built-in options.
---

# Frappe Insights (BI & dashboards)

Insights (`github.com/frappe/insights`) is a Frappe app that adds a self-serve BI tool to a bench: connect data sources, build queries visually or in SQL, turn them into charts, and compose dashboards — all against the site's own database and/or external databases. Its frontend is the same Vue 3 + frappe-ui stack; its data lives in Insights DocTypes on the site.

## When Insights vs built-in reporting

| Need | Use |
|---|---|
| Operational list with filters, printable | ERPNext Report Builder / Query Report (`erpnext` skill, `reports.md`) |
| Fixed KPI cards on a Desk workspace | Desk Dashboard / Number Card |
| Ad-hoc exploration, drag-drop charts, joins across tables, non-developer analysts | **Insights** |
| Analytics over an *external* database or uploaded CSVs | **Insights** (only option in the Frappe stack) |

## How to use this skill

| Task | Reference file |
|---|---|
| Install/update, first-run setup, connecting data sources (site DB, MariaDB, PostgreSQL, CSV), permissions to grant | `references/setup-and-data-sources.md` |
| Building queries — visual builder, joins, filters, aggregation, calculated columns/expressions, raw SQL — and charts | `references/queries-and-charts.md` |
| Dashboards, dashboard filters, sharing (private/public/embed), teams & access control, scheduled alerts | `references/dashboards-and-sharing.md` |

## Ground rules

- **Version matters a lot.** Insights v3 (current) reorganized the UI around *workbooks* (queries + charts + dashboards in one document) and runs query processing through a local DuckDB-backed engine; v2 had standalone Query/Chart/Dashboard documents. Concepts carry over but names and screens differ — check the installed version (`bench version` / app's `__init__.py`) before giving click-path instructions.
- **Read access is not optional.** Queries run with the data source's credentials, not the viewing user's DocType permissions. Anyone who can query a data source can read whatever its DB user can read — scope external DB users to read-only and only the schemas needed, and treat "who can use this data source" as the real permission boundary.
- Insights queries hit the database directly (SQL), bypassing Frappe's ORM — fine for reads, but never point it at a writable credential.
- For heavy queries on a production ERPNext site, prefer a read replica or off-peak scheduled refreshes; a runaway analytical query can starve the site's DB.
- The site DB is MariaDB: DocType tables are `tab<DocType>` (e.g. `tabSales Invoice`), child tables link via `parent`/`parenttype`/`parentfield` — the same schema knowledge as Query Reports applies.
