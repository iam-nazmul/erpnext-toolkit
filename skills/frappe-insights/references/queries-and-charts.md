# Insights Queries & Charts

## Where queries live

- **v3**: inside a **Workbook** — one document holding related queries, charts, and dashboards. Queries are built as a sequence of steps/operations (pick table → join → filter → summarize → sort/limit), each step visible and editable, like a lightweight data pipeline.
- **v2**: standalone Query documents with a single builder screen.

Either way, a query = data source + table(s) + transformations, producing a result table that charts consume.

## Visual query builder

Typical flow for "monthly sales by territory" on an ERPNext site:

1. **Table**: `tabSales Invoice` (Insights shows it by DocType label).
2. **Filter**: `docstatus = 1`, `posting_date` within the period. Add `is_return = 0` if returns should be excluded.
3. **Summarize / aggregate**: sum of `base_grand_total` (company currency — prefer `base_*` fields over `grand_total` in multi-currency setups), grouped by `territory` and by `posting_date` truncated to month (date granularity is an option on the group-by).
4. **Sort & limit** for the preview.

Joins: add the second table and pick the join keys — e.g. `tabSales Invoice Item`.`parent` = `tabSales Invoice`.`name` for line-level analysis (left/inner join selectable). Item-level revenue: sum `base_net_amount` from the child table, grouped by `item_group` from a further join to `tabItem`.

Gotchas that produce wrong numbers:

- Forgetting `docstatus = 1` (drafts/cancelled inflate totals).
- Summing header `base_grand_total` after joining a child table (row multiplication — aggregate the child, or query header only).
- Mixing `grand_total` (document currency) across currencies instead of `base_*`.

## Calculated columns & expressions

Both versions support expression columns with a spreadsheet-like function library (conditionals, date truncation/extraction, string ops, arithmetic). Examples of the style:

```
if_else(docstatus == 1, base_grand_total, 0)
month(posting_date)                      // or date truncation via the column's granularity setting
base_net_total - base_total_taxes_and_charges
```

Function names/syntax differ between v2 and v3 (v3's engine exposes an ibis/DuckDB-flavored function set) — use the in-app expression help/autocomplete as the authority rather than guessing; it lists exactly what the installed version supports. Measures vs dimensions: numeric aggregates you sum/average are measures; group-by columns are dimensions — v3's summarize step makes the distinction explicit.

## SQL queries

Every data source can be queried in raw SQL when the builder gets in the way:

- Site DB / external DBs: write the native SQL dialect (MariaDB/Postgres). Backtick table names with spaces: `` SELECT territory, SUM(base_grand_total) FROM `tabSales Invoice` WHERE docstatus = 1 GROUP BY territory ``.
- A SQL query's result is a table like any other — it can be charted and (v3) used as the input of further builder steps, which is the escape hatch for gnarly subqueries.
- Keep SQL read-only; Insights is not the place for UPDATEs even where credentials would allow it.

## Charts

Charts are built on a query's result: pick chart type, then map result columns to the chart's axes/series. Types across versions: **Number** (big KPI, with optional delta vs previous period), **Line / Area** (time series), **Bar / Row**, **Pie / Donut**, **Table / Pivot**, **Funnel**, **Scatter**, plus mixed-axis variants in v3. Mapping rules of thumb:

- Time series: date-granularity dimension on X, one or more measures on Y; add a second dimension as the series/legend for a breakdown.
- Number cards: query should return one row (or the latest row); use the delta option against the prior period instead of computing % change in the query.
- Table charts are the debugging tool — when a chart looks wrong, view the result as a table first; the error is almost always in the query, not the chart.

Charts live with their query (v3 workbook) and are what dashboards embed — see `dashboards-and-sharing.md`.
