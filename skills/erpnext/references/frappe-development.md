# Frappe Development (custom apps, DocTypes, hooks)

For customizations that ship as code: version-controlled, deployable, testable. For quick tweaks on a hosted instance without deploys, see `low-code-customization.md` instead.

## Bench essentials

```bash
bench new-app my_app                          # scaffold an app
bench --site mysite.local install-app my_app  # install on a site
bench --site mysite.local migrate             # apply schema/patches after pulling changes
bench --site mysite.local console             # IPython with frappe initialized
bench --site mysite.local execute my_app.utils.fn   # run one function
bench --site mysite.local set-config developer_mode 1
bench build && bench restart                  # rebuild assets / reload workers
bench --site mysite.local backup --with-files

# My sites
bench --site bestlifehearingcare.com backup --with-files
bench --site demo.bestlifehearingcare.com backup --with-files
bench --site crm-demo.glascutr.com backup --with-files
bench --site bizboostpos.glascutr.com backup --with-files

bench update                                  # pull + migrate + build all apps (careful in prod)
```

`developer_mode 1` is required for DocType edits made in the UI to be written to your app's source as JSON (so they can be committed). Without it, schema changes land only in the database.

In `bench console`, always end write sessions with `frappe.db.commit()` — console transactions are not auto-committed.

## App layout

```
my_app/
├── my_app/
│   ├── hooks.py                 # integration points with the framework
│   ├── modules.txt
│   ├── patches.txt              # migration entries, run in order by bench migrate
│   ├── my_module/
│   │   └── doctype/
│   │       └── project_task/
│   │           ├── project_task.json    # schema (generated from UI)
│   │           ├── project_task.py      # server controller
│   │           └── project_task.js      # form client script
│   └── public/                  # static assets
└── pyproject.toml
```

## Creating a DocType

With developer mode on, create via UI (DocType list → New) and pick your app's module; the JSON/py/js files are generated in the app. Key schema decisions:

- **Naming**: field-based (`field:customer_name`), series (`PT-.YYYY.-.####`), autoincrement, or prompt.
- **Is Submittable**: adds docstatus lifecycle. Only for transactional docs.
- **Is Child Table**: row type embedded in a parent via a Table field.
- **Track Changes**: keeps a version history — usually yes.
- Field types worth knowing: Link (FK), Dynamic Link (FK whose target DocType is another field), Table, Table MultiSelect, Select, Data, Text/Small Text, Currency/Float/Int, Date/Datetime, Check, Attach, JSON.

## Controller hooks (`project_task.py`)

```python
import frappe
from frappe.model.document import Document

class ProjectTask(Document):
    def validate(self):            # before every save
        if self.end_date and self.end_date < self.start_date:
            frappe.throw("End date cannot precede start date")

    def before_save(self): ...
    def on_update(self): ...       # after save committed to db object
    def before_submit(self): ...
    def on_submit(self): ...       # side effects: create GL entries, notify, etc.
    def on_cancel(self): ...       # must undo on_submit side effects
    def on_trash(self): ...
```

Lifecycle order on save: `before_validate → validate → before_save → (db write) → on_update`. On submit: `before_submit → (docstatus=1) → on_submit`.

## The ORM in one page

```python
doc = frappe.get_doc("Sales Invoice", "ACC-SINV-2026-00042")   # full doc + children
doc.append("items", {"item_code": "X", "qty": 1, "rate": 10})
doc.save()            # runs validations
doc.submit(); doc.cancel(); doc.delete()

frappe.new_doc("ToDo")                                # unsaved instance
frappe.get_all("Item", filters={"disabled": 0},
               fields=["name", "item_group"],
               order_by="modified desc", limit=20)     # no permission check
frappe.get_list(...)                                   # same but permission-checked
frappe.db.get_value("Customer", "Acme", "credit_limit")
frappe.db.set_value("Customer", "Acme", "credit_limit", 5000)  # skips validate! use sparingly
frappe.db.exists("Item", {"item_code": "X"})
frappe.db.sql("select ... where name=%s", (name,), as_dict=True)  # read-only analysis
frappe.throw(_("msg"))   # abort with user-facing error (rolls back)
frappe.msgprint(_("msg"))
```

`frappe.db.set_value` and `doc.db_set` bypass validation and controller hooks — fine for flags/timestamps, wrong for anything with business logic.

Request transactions auto-commit on success and roll back on exception; only call `frappe.db.commit()` yourself in long-running jobs/console.

## hooks.py — the integration points

```python
# React to events on any DocType (including core ones you don't own)
doc_events = {
    "Sales Invoice": {
        "on_submit": "my_app.events.invoice_submitted",
        "validate": "my_app.events.invoice_validate",
    },
    "*": {"on_update": "my_app.events.audit_log"},
}

# Scheduled jobs
scheduler_events = {
    "daily": ["my_app.tasks.daily_digest"],
    "cron": {"*/15 * * * *": ["my_app.tasks.sync_orders"]},
}

# Ship customizations (Custom Fields, Property Setters) with the app
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My App"]]},
]

# Override a core controller
override_doctype_class = {"Sales Invoice": "my_app.overrides.CustomSalesInvoice"}

# Extend core forms with your JS
doctype_js = {"Sales Invoice": "public/js/sales_invoice.js"}
```

Event handler signature: `def invoice_submitted(doc, method): ...`

Export fixtures with `bench --site mysite export-fixtures`; they import automatically on `migrate`.

## Whitelisted API methods

```python
@frappe.whitelist()                       # callable via /api/method/... and frappe.call
def get_dashboard_data(project):
    frappe.has_permission("Project", doc=project, throw=True)
    return {...}

@frappe.whitelist(allow_guest=True)       # public endpoint — validate everything
def public_endpoint(): ...
```

Always enforce permissions inside whitelisted methods; whitelisting only controls *callability*, not authorization.

## Background jobs

```python
frappe.enqueue("my_app.tasks.heavy_sync", queue="long", timeout=3600, order_id=order_id)
```

Queues: short/default/long. Jobs run in workers (`bench worker`); check RQ status at `/app/background_jobs` or `bench --site mysite doctor`.

## Patches (data migrations)

Add to `patches.txt`: `my_app.patches.v1_0.backfill_status`. Each entry runs once per site on `bench migrate`:

```python
def execute():
    frappe.db.sql("""update `tabProject Task` set status='Open' where status is null""")
```

## Testing

```python
# my_app/my_module/doctype/project_task/test_project_task.py
from frappe.tests.utils import FrappeTestCase

class TestProjectTask(FrappeTestCase):
    def test_date_validation(self):
        ...
```

Run: `bench --site mysite run-tests --app my_app` (site needs `allow_tests`; use a dedicated test site).

## Debugging checklist

- Error Log DocType (`/app/error-log`) captures unhandled server exceptions with tracebacks.
- `bench --site mysite console` to reproduce interactively; `frappe.set_user("user@x.com")` to test as a user.
- `bench --site mysite show-config`, `bench doctor` for infra issues.
- Browser devtools network tab shows the failing `frappe.call` and its `_server_messages`.
- After changing Python: `bench restart` (or auto-reload in `bench start` dev mode). After JS/CSS: `bench build --app my_app`.
