# ERPNext Business Workflows

The document chains that automations and integrations must respect. Each arrow is usually created via "mapped" creation (the Create button on the source doc, or `make_*` functions in code) so references and quantities flow correctly.

## Selling cycle

```
Lead → Opportunity → Quotation → Sales Order → Delivery Note → Sales Invoice → Payment Entry
```

- Sales Order (SO) is the commitment; it reserves stock intent and drives billing/delivery status (`per_delivered`, `per_billed`).
- Delivery Note (DN) posts the stock movement (Stock Ledger Entry); Sales Invoice (SI) posts accounting (GL Entry: Debtors ↔ Income).
- Shortcuts exist: SI can be created directly from SO ("bill before delivery"), or with **Update Stock** checked the SI itself moves stock (common in retail/POS — then no DN).
- Mapped creation in code:

```python
from erpnext.selling.doctype.sales_order.sales_order import make_delivery_note, make_sales_invoice
dn = make_delivery_note("SAL-ORD-2026-00031")
dn.insert(); dn.submit()
```

## Buying cycle

```
Material Request → Request for Quotation → Supplier Quotation → Purchase Order → Purchase Receipt → Purchase Invoice → Payment Entry
```

Mirror of selling: Purchase Receipt moves stock in, Purchase Invoice posts Creditors ↔ Expense/Stock Received But Not Billed.

## Stock

- **Item** master: `is_stock_item` decides whether it hits the stock ledger. Variants come from a template item + attributes. `has_batch_no` / `has_serial_no` add batch/serial tracking (v15 uses the Serial and Batch Bundle mechanism on transactions).
- **Warehouse** is a tree; stock lives at (item, warehouse) and is valued FIFO or Moving Average per item.
- **Stock Entry** handles non-sales/purchase movements — purposes: Material Receipt, Material Issue, Material Transfer, Manufacture, Repack.
- **Stock Reconciliation** sets absolute qty/valuation (opening stock, audits).
- Never insert Stock Ledger Entries or Bins directly; submit the transaction documents and let ERPNext post them.
- Read stock via the Stock Balance / Stock Ledger reports or Bin (`frappe.db.get_value("Bin", {"item_code": ..., "warehouse": ...}, "actual_qty")`).

## Accounting

- Chart of Accounts is a tree per Company; ledger postings are **GL Entry** rows, always created by submitting documents (SI, PI, Payment Entry, Journal Entry) — never by hand.
- **Payment Entry** records receive/pay and allocates against invoices (reconciliation). **Journal Entry** is the general-purpose instrument.
- Multi-company: almost every transactional doc carries `company`; masters like Customer are shared, but accounts/dimensions are per-company.
- Cancelling a submitted doc reverses its GL/stock entries; **amend** creates `NAME-1` as a new draft.
- Period closing, fiscal years, and cost centers (plus custom Accounting Dimensions) segment reporting.

## HR (separate HRMS app since v14)

Employee → Attendance/Leave Application → Salary Structure Assignment → Payroll Entry → Salary Slip. If HR doctypes are missing, the `hrms` app isn't installed.

## Manufacturing

Item (with BOM) → BOM → Work Order → Stock Entry (Material Transfer for Manufacture, then Manufacture) → finished goods in stock. Job Cards track operations.

## Statuses worth knowing

- Sales Order: Draft, To Deliver and Bill, To Bill, To Deliver, Completed, Cancelled, Closed, On Hold.
- Sales/Purchase Invoice: Unpaid, Partly Paid, Paid, Overdue, Return, Credit Note Issued...
- Status is mostly derived — set by the system from docstatus + linked docs. Don't write to `status` directly; perform the action that changes it.

## Automation patterns

- **New order intake from an external system**: create Customer if missing (check by `customer_name` or a custom external-ID field), create Items or map SKUs via Item Code, POST Sales Order, submit. Idempotency: store the external order ID in a custom field and check `frappe.db.exists` before creating.
- **Sync out on changes**: Webhook on Sales Invoice on_submit → your endpoint; or scheduler polling `modified > last_sync` (every doc has `modified`, indexed).
- **Returns**: `is_return = 1` documents (Credit Note = return Sales Invoice, `return_against` links the original).
- **Pricing**: rates resolve via Price List + Item Price + Pricing Rules; when creating orders via API, omit `rate` to let ERPNext fetch it, or send `rate` explicitly to override.
- **Taxes**: pass a `taxes_and_charges` template name or explicit `taxes` child rows; totals recompute on save.
