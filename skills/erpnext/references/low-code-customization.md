# Low-code Customization (no deploy needed)

Everything here lives in the database and works on hosted instances (Frappe Cloud, managed hosting) where you can't push code. It's the right tool for small tweaks; once logic grows, migrate it into a custom app (see `frappe-development.md`).

On self-hosted setups, Server Script must be enabled: `bench --site mysite set-config server_script_enabled 1` (Frappe Cloud has it on by default).

## Custom Fields

Add fields to any DocType without touching its source: **Customize Form** (per-DocType view) or the Custom Field list directly. Custom fieldnames get no special prefix by default — adopt a convention like `custom_` (v15 does this automatically for fields added via Customize Form).

Customize Form also adjusts properties of existing fields (hidden, read-only, label, reqd, options) — stored as **Property Setters**. Core schema is untouched, so upgrades are safe.

Export both to a custom app later via fixtures if you outgrow low-code.

## Client Scripts (browser-side)

DocType: **Client Script**. Runs on forms (and list views in v15). Classic form events:

```javascript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button('Create Task', () => { ... });
        }
    },
    customer(frm) {                      // field-change event
        frm.set_value('territory', null);
    },
    validate(frm) {
        if (!frm.doc.po_no) {
            frappe.throw(__('Customer PO required'));
        }
    }
});

// Child table events: (parent DocType stays in the on() of the child DocType name)
frappe.ui.form.on('Sales Order Item', {
    qty(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        frappe.model.set_value(cdt, cdn, 'amount', row.qty * row.rate);
    },
    items_add(frm, cdt, cdn) { ... }     // row added
});
```

Useful client APIs: `frm.set_value`, `frm.set_query('customer', () => ({filters: {...}}))` (filter link dropdowns), `frm.toggle_display`, `frm.set_df_property`, `frappe.call({method, args, callback})`, `frappe.db.get_value` (async), `frappe.msgprint`, `frappe.throw`.

## Server Scripts (sandboxed Python)

DocType: **Server Script**. Script types:

**1. DocType Event** — pick DocType + event (Before Save, After Save, Before Submit, After Submit, Before Cancel, After Cancel, Before Delete...). The document is available as `doc`:

```python
# Before Save on Sales Order
if doc.discount_percentage > 20 and "Sales Manager" not in frappe.get_roles():
    frappe.throw("Discounts above 20% need a Sales Manager")
```

**2. API** — exposes `/api/method/<script-api-method-name>`:

```python
# api method name: get_open_orders
customer = frappe.form_dict.customer
frappe.response["message"] = frappe.get_all(
    "Sales Order",
    filters={"customer": customer, "status": "To Deliver and Bill"},
    fields=["name", "grand_total"],
)
```

**3. Scheduler Event** — cron-style scheduled scripts.

**4. Permission Query** — append SQL conditions to list queries for row-level access.

The sandbox exposes a restricted `frappe` namespace: `frappe.get_doc`, `frappe.new_doc`, `frappe.get_all`/`get_list`, `frappe.db.get_value`/`set_value`/`exists`/`count`, `frappe.throw`/`msgprint`, `frappe.session.user`, `frappe.enqueue`, plus `log()` for debugging (output lands in the Server Script log). No imports allowed — if you need a library, you need a custom app.

Gotcha: in Before Save scripts, mutate `doc` directly (it will be saved); in After Save/Submit, changing `doc` does nothing — use `frappe.db.set_value` or create other documents.

## Other no-code building blocks

- **Workflow** — multi-state approval chains (states, transitions, allowed roles). Replaces the plain draft→submit flow; the workflow drives docstatus changes.
- **Notification** — email/system/Slack alerts on document events or dates, with Jinja templates and conditions (`doc.grand_total > 100000`).
- **Webhook** — outbound HTTP calls on doc events (see rest-api.md).
- **Auto Repeat** — recurring documents (e.g., monthly invoices).
- **Assignment Rule** — round-robin/load-based auto-assignment.
- **Print Format** — Jinja or drag-drop invoice/document layouts.
- **Data Import** — CSV/XLSX bulk insert/update with template download; for updates you must include the `ID` (name) column. Test with a handful of rows first; imports run validations, so they're safe but slow on huge files (use the "Submit after import" checkbox deliberately).
- **Dashboard / Dashboard Chart / Number Card** — no-code analytics on top of DocTypes or report data.
