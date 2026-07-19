# ERPNext REST API

Every DocType automatically gets a REST interface. Base URL is the site URL, e.g. `https://erp.example.com`.

## Authentication

**Token auth (recommended for integrations).** Each user can generate an API Key + API Secret (User → Settings → API Access). Send on every request:

```
Authorization: token <api_key>:<api_secret>
```

The request runs with that user's roles and permissions. Create a dedicated integration user with only the roles it needs — do not use Administrator for integrations.

**Session auth (browser-like flows).**

```http
POST /api/method/login
Content-Type: application/json

{"usr": "user@example.com", "pwd": "password"}
```

Returns a `sid` cookie; send it on subsequent requests. Use token auth unless you specifically need a session.

**OAuth 2.0** is also available (Frappe is an OAuth provider) for third-party apps acting on behalf of users.

## CRUD on documents

All endpoints are under `/api/resource/<DocType>`. URL-encode spaces in DocType names (`Sales%20Invoice`).

### List documents

```
GET /api/resource/Customer
GET /api/resource/Sales Invoice?fields=["name","customer","grand_total"]&filters=[["status","=","Unpaid"]]&limit_page_length=50&limit_start=0&order_by=posting_date desc
```

- `fields` — JSON array of field names. Default is only `name`. Use `["*"]` sparingly (slow).
- `filters` — JSON array of `[fieldname, operator, value]` triples. Operators: `=`, `!=`, `>`, `<`, `>=`, `<=`, `like`, `not like`, `in`, `not in`, `between`, `is` (value `"set"`/`"not set"`).
- `or_filters` — same shape, OR'd together.
- `limit_page_length` — default 20; `0` means no limit (dangerous on big tables).
- Child table fields can be fetched with dotted syntax in `fields`, e.g. `["name","items.item_code"]` — this returns one row per child row (a join).

Example filters:

```
filters=[["posting_date","between",["2026-01-01","2026-06-30"]],["docstatus","=",1]]
filters=[["customer_name","like","%acme%"]]
```

### Read one document

```
GET /api/resource/Sales Invoice/ACC-SINV-2026-00042
```

Returns the full document **including child tables** as nested arrays.

### Create

```http
POST /api/resource/Sales Invoice
Content-Type: application/json

{
  "customer": "Acme Corp",
  "posting_date": "2026-07-05",
  "items": [
    {"item_code": "WIDGET-001", "qty": 10, "rate": 25.0}
  ]
}
```

Response contains the created doc with its generated `name`. Missing mandatory fields return a 417 with a validation message. Link field values must be existing document names.

### Update

```http
PUT /api/resource/Customer/Acme Corp
{"credit_limit": 50000}
```

Partial updates are fine. **Child tables are replaced wholesale**: if you send `items`, the list you send becomes the entire child table. To edit one row, GET the doc, modify the array, and PUT it back (include each existing row's `name` to preserve rows).

Updating a submitted doc (docstatus 1) fails unless the field allows on submit.

### Delete

```
DELETE /api/resource/Customer/Acme Corp
```

Fails if the document is linked from other documents or submitted.

## Submit, cancel, and other actions

Submission is a method call, not a plain field update:

```http
POST /api/method/frappe.client.submit
{"doc": {<full document JSON as returned by GET>}}

POST /api/method/frappe.client.cancel
{"doctype": "Sales Invoice", "name": "ACC-SINV-2026-00042"}
```

A common pattern: POST to create (docstatus 0), GET the full doc, then `frappe.client.submit` it.

## RPC-style method calls

Any whitelisted Python method is callable:

```
GET  /api/method/frappe.auth.get_logged_user
POST /api/method/erpnext.stock.utils.get_stock_balance   (if whitelisted in your version)
POST /api/method/myapp.api.my_function                    (your own @frappe.whitelist() method)
```

- GET for read-only methods, POST for anything that writes (POST also gets CSRF exemption with token auth).
- Arguments go as query params (GET) or JSON body (POST).
- Return value is wrapped in `{"message": ...}`.

Useful built-ins under `frappe.client`: `get`, `get_list`, `get_value`, `set_value`, `insert`, `submit`, `cancel`, `delete`, `get_count`.

## File uploads

```http
POST /api/method/upload_file
Content-Type: multipart/form-data

file: <binary>
doctype: Sales Invoice          (optional — attach to a doc)
docname: ACC-SINV-2026-00042
is_private: 1
```

Returns a File document; its `file_url` can be set on Attach fields.

## Errors and gotchas

| Symptom | Likely cause |
|---|---|
| 403 Forbidden | User lacks role permission on the DocType, or method not whitelisted |
| 417 Expectation Failed | Validation error — body contains `_server_messages` with the human-readable reason |
| 404 on a doc you can list | Row-level User Permissions hiding it, or name typo (names are case-sensitive) |
| 409 / TimestampMismatchError | Doc modified since you fetched it — re-GET and resend with fresh `modified` |
| Duplicate entry | Naming collision — you supplied a `name`/ID that exists |
| CSRF token error | Session auth on POST without token — use token auth or fetch CSRF token |

Rate limiting can be configured per site (`rate_limit` in site_config.json); back off on 429.

## Webhooks (ERPNext → you)

For push instead of poll, create a **Webhook** document in the UI: pick DocType + event (on_update, on_submit, etc.), set the target URL, choose fields or full doc JSON, and optionally a shared secret (sent as `X-Frappe-Webhook-Signature`, HMAC-SHA256 of the payload). Conditions support Python expressions like `doc.status == "Paid"`.
