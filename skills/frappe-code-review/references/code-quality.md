# Frappe Code Quality & Correctness

Bugs that aren't security holes but corrupt data, break on submit, or fall over under load. Review after the security pass.

## Controllers & document lifecycle

- **Right hook for the job.** `validate` (before save, every save) for validation/derivation; `before_save`/`after_insert` for one-time setup; `on_submit`/`on_cancel` for ledger-affecting actions; `on_update_after_submit` only for Allow-on-Submit fields. Doing ledger postings in `validate` (runs on drafts) is a bug; doing validation in `on_submit` (too late) is another.
- **`docstatus` awareness.** Code that edits a doc must respect that submitted docs (docstatus 1) are immutable except Allow-on-Submit fields. Look for `.save()` on submitted docs, or logic that assumes a doc is always draft.
- **`on_cancel` must undo `on_submit`.** Every side effect created on submit (linked docs, ledger entries, status changes) needs a matching reversal on cancel, or cancellation leaves orphans.
- **Idempotency & re-entrancy.** `doc_events` and scheduled jobs can run more than once; creating a linked document without checking for an existing one produces duplicates.
- **Naming/`autoname`** collisions and race conditions on custom naming series.

## Transactions & the database

- **Explicit `frappe.db.commit()` inside a request is almost always wrong.** Frappe commits at the end of the request; a mid-request commit defeats rollback-on-error and can persist half-finished state. Legitimate only in long-running jobs/migrations with clear intent. Flag every one.
- **`frappe.db.rollback()`** in the wrong place discards unrelated pending writes.
- **Reading your own uncommitted writes**: `db.set_value` then `db.get_value` in the same request is fine; mixing ORM doc state with direct `db.*` writes causes stale-in-memory bugs (`doc.field` still old after a `db.set_value`).
- **`db_set` vs `set_value` vs saving the doc**: `db.set_value`/`db_set` skip validation and hooks — appropriate for a single denormalized/status field, wrong when validation should run.

## ORM misuse & performance

- **N+1 queries**: `frappe.get_doc` or `db.get_value` inside a loop over rows. Batch with a single `frappe.get_all(..., filters={"name": ["in", names]})` and index the result.
- **`frappe.get_all` without `limit`** on user-facing paths can pull the whole table; paginate.
- **`get_doc` when you need one field** — use `db.get_value`; `get_doc` loads the whole doc + children.
- **Missing indexes** on custom fields used in frequent filters/joins (`search_fields`, report filters). Check `frappe.db.add_index` / DocType "index" flags.
- **Whitelisted methods returning large payloads** — cost and exposure; return projections and paginate.
- **Repeated `frappe.get_meta` / settings reads** in loops — cache outside the loop.

## Correctness traps specific to ERPNext

- **Currency**: mixing `grand_total` (document currency) and `base_grand_total` (company currency); rounding via `flt(x, precision)` not raw floats; multi-currency conversion factors.
- **Ledger integrity**: never write GL Entry / Stock Ledger Entry directly — go through the posting APIs. Direct manipulation is both a correctness and an accounting-integrity failure.
- **`flt` / `cint` / `getdate`**: raw arithmetic on possibly-None fields; date string vs datetime comparisons across timezones.
- **Fetching linked-doc values at write time** vs relying on stale fetched fields.

## Structure, hooks & app hygiene

- **`hooks.py`** review: overly broad `doc_events` (`"*"` on `on_update` doing heavy work), scheduler entries with wrong frequency, `override_doctype_class` / `override_whitelisted_methods` that silently change core behavior.
- **Patches/migrations** (`patches.txt`): idempotent? guarded against re-run? touching data with validation-skipping writes?
- **Fixtures**: exporting more than intended (Custom Fields, Property Setters) or user data.
- **Error handling**: bare `except:` swallowing failures; `frappe.log_error(title=...)` used correctly (title ≤ 140 chars or it truncates); not logging silently where a `throw` is warranted.
- **Tests**: presence of `test_*.py`; whether new controllers/whitelisted methods have coverage of the permission and validation paths — the parts most likely to regress.

## Reporting quality findings

Lead with correctness/data-integrity issues (silent corruption > crash > perf > style). Give the failure scenario ("on cancel, the linked Payment Entry is left submitted, so the invoice can't be re-amended") and the specific fix, with `file:line`.
