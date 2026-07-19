# ERPNext Toolkit — Claude Code Plugin

All-in-one plugin for working with ERPNext / Frappe Framework from Claude Code: bundled skills, slash commands, specialized agents, and a live MCP connection to your ERPNext instance.

## What's inside

| Component | Items |
|---|---|
| **Skills** | `erpnext` (API, DocTypes, reports, workflows, troubleshooting), `frappe-frontend` (Vue 3 + frappe-ui + Tailwind portals/PWAs), `frappe-ui` (component library, theming, composables), `frappe-insights` (BI queries, charts, dashboards), `frappe-code-review` (Frappe-specific code & security review) |
| **Commands** | `/erp-query` (read data), `/erp-doctype` (design DocTypes), `/erp-report` (build reports), `/erp-fix` (troubleshoot errors) |
| **Agents** | `erpnext-developer` (code tasks), `erpnext-analyst` (business data analysis, read-only) |
| **MCP** | Live connection to your ERPNext instance via `@casys/mcp-erpnext` (community server, 120+ tools) |

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

### 3. Verify

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
