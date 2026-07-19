# Insights Dashboards, Sharing & Access Control

## Building dashboards

A dashboard is a grid of chart widgets plus text/filter widgets. In v3 dashboards live inside a workbook next to the queries/charts they display; in v2 they're standalone documents that pull in any shared chart.

Practices that keep dashboards useful:

- Lead with 3–5 **Number** cards (the KPIs), then trends (line/area), then breakdowns (bar/table). One question per chart.
- Name charts as answers ("Revenue by Territory — last 12 months"), not table names.
- Set an **auto-refresh / cache duration** appropriate to the data: heavy queries on production DBs should refresh on a schedule, not on every viewer load.

## Dashboard filters

Filter widgets bind one user-facing control (date range, select, text) to columns of the underlying queries — link the filter to the matching column on each chart's query so one date picker drives the whole dashboard. Date-range + one or two categorical filters (Company, Territory, Warehouse) cover most ERPNext dashboards. Filters are the reason to keep raw-SQL queries few: builder queries expose their columns for linking cleanly; SQL queries may need explicit variables/placeholders depending on version.

## Sharing

Scopes, from tightest to loosest:

1. **Private** — visible to the creator (and admins).
2. **Team-shared** — members of teams with access (the normal mode; see below).
3. **Public link** — anyone with the URL, no login. The underlying data becomes effectively public; never enable it on dashboards containing customer-identifiable or financial detail without sign-off.
4. **Embed** — public dashboards expose an iframe embed snippet for external sites/portals. Same warning as public links, plus check `X-Frame-Options`/CSP on the site if the iframe refuses to render.

Sharing a dashboard does not require viewers to have query-building rights — viewers see results, builders see queries.

## Teams & access control

Insights layers its own model on top of site login:

- **Roles**: Insights Admin (everything: sources, settings, all teams) vs Insights User (build within granted access).
- **Teams** group users and are granted access to **data sources** (and table-level restriction within a source, per version). A user can only query tables their team can reach — this, plus read-only DB credentials, is the entire security model. DocType-level Frappe permissions do *not* apply to Insights queries.

Practical setup for a company: an `Analysts` team with the site DB source; a `Finance` team additionally granted the accounting warehouse source; public sharing disabled by policy except for a vetted TV-dashboard.

## Alerts & scheduled delivery

Insights supports alerts on query results — a condition checked on a schedule (cron/interval) that sends a notification (email; Telegram in some versions) when it trips, e.g. "daily sales below X" or "negative stock rows > 0". Configure from the query/chart's alert option. Under the hood these run via the site's scheduler — if alerts never fire, check `bench doctor` for scheduler status like any other Frappe scheduled job.

## Operational notes

- Everything (sources, workbooks/queries, dashboards, teams) is stored as DocTypes on the site — normal backups, `bench migrate` on update, and fixtures/export for promoting dashboards between sites are possible, though dashboards are usually rebuilt rather than migrated.
- Slow dashboard? Find the slow query (each widget shows load state), then optimize it at the query level — add date filters, pre-aggregate, or move the source to a replica. The dashboard layer adds almost no overhead itself.
- Insights' REST API surface (Insights DocTypes via `/api/resource/...`) exists but is internal-ish; for programmatic analytics prefer querying the DB or the documented Frappe API rather than scripting Insights internals.
