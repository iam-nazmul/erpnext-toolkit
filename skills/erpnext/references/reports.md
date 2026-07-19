# Reports in ERPNext

Three tiers — pick the simplest that works:

| Type | Built with | When |
|---|---|---|
| Report Builder | UI only | Column picks + filters on one DocType (+ linked fields) |
| Query Report | SQL | Joins/aggregations, read-only, no Python needed |
| Script Report | Python | Computed logic, multi-step data prep, charts, custom filters |

Query and Script Reports are documents of DocType **Report**. On hosted instances, Query Reports can be written in the UI by System Managers; Script Reports require developer mode / a custom app (their code lives in files) — the low-code alternative is a Server Script of type API feeding a page or chart.

## Query Report

Report Type = "Query Report", pick the reference DocType (controls who can see it via that DocType's permissions), and write SQL. Filter values are injected as `%(fieldname)s`:

```sql
select
    si.name as "Invoice:Link/Sales Invoice:160",
    si.customer as "Customer:Link/Customer:160",
    si.posting_date as "Date:Date:100",
    si.grand_total as "Total:Currency:120",
    si.outstanding_amount as "Outstanding:Currency:120"
from `tabSales Invoice` si
where si.docstatus = 1
  and si.posting_date between %(from_date)s and %(to_date)s
order by si.posting_date desc
```

Column headers encode formatting: `Label:Fieldtype/Options:Width`. Table names are `tab<DocType>` with backticks. Filters for query reports are defined in the report's Filters child table (fieldname, label, fieldtype, options, default).

## Script Report

Created with developer mode on (files land in your app under `report/<report_name>/`). Two files matter:

**`my_report.py`** — must define `execute`:

```python
import frappe

def execute(filters=None):
    filters = frappe._dict(filters or {})

    columns = [
        {"label": "Item", "fieldname": "item_code", "fieldtype": "Link",
         "options": "Item", "width": 180},
        {"label": "Qty Sold", "fieldname": "qty", "fieldtype": "Float", "width": 110},
        {"label": "Revenue", "fieldname": "revenue", "fieldtype": "Currency", "width": 130},
    ]

    data = frappe.db.sql("""
        select sii.item_code,
               sum(sii.stock_qty) qty,
               sum(sii.base_net_amount) revenue
        from `tabSales Invoice Item` sii
        join `tabSales Invoice` si on si.name = sii.parent
        where si.docstatus = 1
          and si.posting_date between %(from_date)s and %(to_date)s
        group by sii.item_code
        order by revenue desc
    """, filters, as_dict=True)

    chart = {
        "data": {
            "labels": [d.item_code for d in data[:10]],
            "datasets": [{"values": [d.revenue for d in data[:10]]}],
        },
        "type": "bar",
    }

    report_summary = [
        {"label": "Total Revenue", "value": sum(d.revenue for d in data),
         "datatype": "Currency"},
    ]

    return columns, data, None, chart, report_summary
```

Return signature: `(columns, data)` minimally; optionally `(columns, data, message, chart, report_summary, skip_total_row)`.

**`my_report.js`** — filter definitions:

```javascript
frappe.query_reports["My Report"] = {
    filters: [
        {fieldname: "from_date", label: __("From Date"), fieldtype: "Date",
         default: frappe.datetime.add_months(frappe.datetime.get_today(), -1), reqd: 1},
        {fieldname: "to_date", label: __("To Date"), fieldtype: "Date",
         default: frappe.datetime.get_today(), reqd: 1},
        {fieldname: "customer", label: __("Customer"), fieldtype: "Link",
         options: "Customer"},
    ],
};
```

## Practical notes

- **Respect docstatus**: nearly every transactional report needs `docstatus = 1`, or you'll count drafts and cancelled docs.
- Financial figures: prefer `base_*` fields (company currency) when aggregating across currencies; GL-accurate numbers come from `tabGL Entry`, stock from `tabStock Ledger Entry` (`actual_qty`, `stock_value_difference`).
- Reports can be run over REST: `GET /api/method/frappe.desk.query_report.run?report_name=My%20Report&filters={"from_date":"2026-01-01","to_date":"2026-06-30"}`.
- Any report can be exported (CSV/XLSX) from the UI, scheduled by email via **Auto Email Report**, or embedded as a Dashboard Chart source.
- Before building anything, check the standard reports (Accounts Receivable, General Ledger, Stock Balance, Sales Analytics, Profit & Loss…) — many "custom report" requests are a standard report plus a filter.
