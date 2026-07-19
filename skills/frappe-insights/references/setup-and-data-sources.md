# Insights Setup & Data Sources

## Installation

Insights is a normal Frappe app:

```bash
bench get-app insights          # or: bench get-app https://github.com/frappe/insights
bench --site mysite.local install-app insights
bench --site mysite.local migrate
```

Open it at `https://<site>/insights` (SPA served from the app's `www/` folder, same session as the site). On Frappe Cloud it's installable from the marketplace. Dev setup for hacking on Insights itself follows the standard frappe-ui pattern: `cd apps/insights/frontend && yarn dev` proxied to the site (see the frappe-frontend skill).

Version check: v3 is the current line (workbooks, DuckDB-backed query engine); v2 sites in the wild still exist. `bench --site mysite.local console` → `import insights; insights.__version__` if unsure.

## Who can use it

Access is controlled by Insights' own roles + teams, not by generic Desk roles:

- **Insights Admin** — manage data sources, all teams, settings.
- **Insights User** — build queries/dashboards on sources their team can access.

Grant one of these roles to the User; a System Manager typically gets admin implicitly. Portal/Website users are not the audience — Insights is an internal analyst tool. Fine-grained "who sees which source/table" is handled via **Teams** (see `dashboards-and-sharing.md`).

## Data sources

### The site's own database

Insights ships with the current site's database preconfigured as a data source (commonly named "Site DB"/"Frappe Site"). That's the zero-setup path for ERPNext analytics. Remember the schema rules: tables are `` `tabSales Invoice` ``, child tables carry `parent`, dates are strings/datetimes in site timezone, `docstatus` 0/1/2 = draft/submitted/cancelled — filter `docstatus = 1` for real transactional numbers.

### External MariaDB / PostgreSQL

Add via Insights → Data Sources → New: host, port, database, username, password, SSL. Rules:

- Create a **dedicated read-only DB user**, granted `SELECT` only on the schemas/tables needed. Never reuse an application credential.
- The bench server must be able to reach the DB (firewall/VPC); test with `mysql`/`psql` from the server first when a connection fails — most "Insights can't connect" issues are network, not Insights.
- After connecting, Insights syncs the table list; individual tables can be hidden from analysts.

### CSV / file uploads

Upload a CSV (v3: into a workbook / the uploads area) and it becomes a queryable table — Insights stores it and (in v3) queries it through DuckDB. Good for one-off analysis and for joining reference data (targets, budgets, mappings) against live ERPNext tables. Re-upload to refresh; this is not a pipeline — for recurring external data, land it in a proper database instead.

### ERPNext on another site

There's no special connector — treat the other site's MariaDB as an external data source (read-only user on its DB), or replicate into a warehouse DB that Insights reads.

## Performance & safety practices

- Point heavy analytical use at a **read replica** of the production DB where possible; long scans on the primary can degrade the ERPNext site.
- In v3, query results are processed/cached via the local DuckDB engine and result caching — still, limit preview row counts and add date filters early when exploring big tables.
- Back up consideration: Insights queries/dashboards are DocTypes in the site DB, so normal `bench backup` covers them; external source credentials are stored encrypted on the site.
