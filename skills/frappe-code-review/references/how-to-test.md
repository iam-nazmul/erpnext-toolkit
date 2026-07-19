# How to Run the Analysis

Static grep to find candidates, then dynamic testing on a **dev/staging bench** to confirm. Never probe someone else's production.

## Static: grep the attack surface

Run from the app root. These surface the five red flags and most checklist items:

```bash
# Entry points
grep -rn "@frappe.whitelist" .                       # every callable endpoint
grep -rn "allow_guest=True" .                        # unauthenticated endpoints
grep -rn "override_whitelisted_methods\|doc_events\|scheduler_events" hooks.py

# Injection
grep -rEn "frappe\.db\.sql\(.*(f\"|f'|%|\.format\(|\+)" .   # formatted SQL → likely injection
grep -rn "frappe.db.sql" .                                  # all raw SQL, review order_by/fields too

# Permission bypass
grep -rn "ignore_permissions" .
grep -rn "frappe.get_all\|frappe.db.get_value\|frappe.db.get_list" .   # do NOT check perms
grep -rn "\.update(frappe.form_dict\|\.update(kwargs" .                # mass assignment

# Dangerous execution / secrets
grep -rEn "os\.system|subprocess|(^|[^.])eval\(|exec\(|safe_eval" .
grep -rn "requests.get\|requests.post\|urlopen" .           # SSRF candidates
grep -rEn "(password|secret|token|api_key)\s*=\s*['\"]" .   # hardcoded secrets

# Frontend
grep -rn "v-html\|innerHTML\|dangerouslySetInnerHTML" frontend/ src/ 2>/dev/null
```

Triage each hit against `security-checklist.md`. A hit is a *candidate*, not a finding — confirm the input is user-controlled and the guard is absent.

## Static: available tooling

- **Ruff / flake8 / bandit** — `bandit -r app/` catches generic Python security smells (though it doesn't know Frappe's `db.sql` escaping semantics, so combine with the greps above).
- **Semgrep** — write or reuse rules for `frappe.db.sql` string-formatting and `ignore_permissions`; good for repeatable CI gating.
- **Frappe's own linters** — `bench` projects often ship pre-commit (ruff, eslint, prettier); run them to separate style noise from real findings.
- **eslint** on the Vue frontend; look for `no-unsanitized`/`v-html` rules.
- The built-in **`/code-review`** and **`/security-review`** commands for the generic diff pass — this skill adds the Frappe layer on top.

## Dynamic: prove it on a dev bench

Set up an isolated site so you can exercise findings without risk:

```bash
bench new-site review.local
bench --site review.local install-app <the_app>
bench --site review.local add-user attacker@test.com --user-type "System User"   # a low-priv user
bench --site review.local set-admin-password admin
bench start
```

- **Reproduce as a real user.** Get an API key for the low-privilege user, then call the suspect endpoint directly (bypassing the UI):
  ```bash
  curl -s -H "Authorization: token <api_key>:<api_secret>" \
    "http://review.local:8000/api/method/the_app.api.suspect_method?name=SOME-DOC-THEY-SHOULDNT-SEE"
  ```
  If it returns data the user has no permission for → confirmed IDOR/permission bypass.
- **SQL injection**: send a payload param (`' OR '1'='1`, or a time-based `SLEEP(3)`) and observe whether results change / the response is delayed. Confirm in the query log (`bench --site review.local console`, or MariaDB general log) that the input reached SQL unescaped.
- **`bench --site review.local console`** to inspect state, run the method as different users (`frappe.set_user(...)`), and check what permission functions return.
- **Mass assignment**: POST extra fields (`owner`, `docstatus`, an amount field) to a whitelisted create/update method and check whether they stick.
- Watch `bench --site review.local` logs and `logs/` for tracebacks that reveal internals.

## Turning it into a report

For each confirmed finding: severity (per `security-checklist.md`), `file:line`, the exact input that triggers it, the observed impact from the dynamic test, and the minimal fix (parameterize, add `frappe.has_permission`, whitelist fields, remove `ignore_permissions`). Keep unconfirmed candidates in a separate "needs verification" list with the reproduction step to run — don't inflate them to findings. If reporting on a PR, `/code-review --comment` posts inline; otherwise deliver the ranked list in the final message.
