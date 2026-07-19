# ERPNext Toolkit — Claude Code Plugin

All-in-one plugin for working with ERPNext / Frappe Framework from Claude Code: bundled skills, slash commands, specialized agents, and a live MCP connection to your ERPNext instance.

## What's inside

| Component | Items |
|---|---|
| **Skills** | `erpnext` (API, DocTypes, reports, workflows, troubleshooting), `frappe-frontend` (Vue 3 + frappe-ui + Tailwind portals/PWAs), `frappe-ui` (component library, theming, composables), `frappe-insights` (BI queries, charts, dashboards), `frappe-code-review` (Frappe-specific code & security review) |
| **Commands** | `/erp-query` (read data), `/erp-doctype` (design DocTypes), `/erp-report` (build reports), `/erp-fix` (troubleshoot errors) |
| **Agents** | `erpnext-developer` (code tasks), `erpnext-analyst` (business data analysis, read-only) |
| **MCP** | Live connection to your ERPNext instance via `@casys/mcp-erpnext` (community server, 120+ tools) |

## Example use cases

**Query business data** (`/erp-query` or just ask)

- "Show me all outstanding sales invoices over 30 days"
- "Who are our top 10 customers by revenue this quarter?"
- "What's the current stock balance for Item X across all warehouses?"

**Business analysis** (`erpnext-analyst` agent)

- "Analyze our revenue trend over the last 6 months and flag anomalies"
- "Build a sales funnel summary — leads → opportunities → quotations → orders — and find where we lose the most deals"
- "Compare gross margin by item group and tell me which products to discontinue"

**Development** (`erpnext-developer` agent + skills)

- "Create a custom DocType for equipment rental agreements with a child table for rental items" (`/erp-doctype`)
- "Write a server script that auto-applies a 5% discount when order total exceeds 50,000"
- "Add a scheduled job that emails a daily summary of overdue invoices to accounts"

**Reports & dashboards**

- "Build a Script Report showing item-wise profitability with warehouse filters" (`/erp-report`)
- "Set up a Frappe Insights dashboard for monthly sales KPIs"

**Custom frontends** (Vue 3 + frappe-ui)

- "Build a customer portal where clients can view their invoices and payment status"
- "Make a warehouse kiosk PWA for scanning and recording stock entries"

**Troubleshooting & review**

- "I'm getting a ValidationError when submitting this sales order — fix it" (`/erp-fix`)
- "Audit my custom app for permission bypasses and SQL injection" (frappe-code-review skill)

**Operations via chat** (MCP write tools)

- "Create a sales order for Customer ABC: 10 units of ITEM-001, deliver next Friday, then submit it"

## Install

In Claude Code:

```
/plugin marketplace add iam-nazmul/erpnext-toolkit
/plugin install erpnext-toolkit@erpnext-marketplace
```

On install, Claude Code prompts for three values (stored as plugin user config):

- **ERPNext URL** — e.g. `https://your-site.erpnext.com`
- **ERPNext API key** and **API secret** — in ERPNext, go to **User → API Access → Generate Keys**. It is safest to create a dedicated user and give it only the roles it needs.

Requires Node.js ≥ 20 (the MCP server runs via `npx`).

**Local development / testing** — run Claude Code with the plugin loaded from a folder instead:

```bash
claude --plugin-dir /path/to/erpnext-toolkit
```

### Verify

- Run `/erp-query outstanding sales invoices` to check that the MCP connection works
- `/agents` — you should see the two agents listed
- The skills trigger automatically whenever ERPNext/Frappe comes up

## Security notes

- The MCP server can both read and write, according to your API key's permissions — use a least-privilege user on production instances.
- The `erpnext-analyst` agent is read-only by its rules, but the final control is the API user's roles.
- `@casys/mcp-erpnext` is a community project — review it before installing: https://github.com/Casys-AI/mcp-erpnext

## Privacy

The plugin collects nothing and runs entirely on your machine — credentials are stored locally by Claude Code, and data flows only to your own ERPNext instance and to Anthropic (as with all Claude Code usage). Full details: [PRIVACY.md](PRIVACY.md)

## Docs

- Claude Code plugins: https://code.claude.com/docs/en/plugins-reference
- ERPNext: https://docs.frappe.io/erpnext
