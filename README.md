# ERPNext Toolkit — Claude Code Plugin

All-in-one plugin for working with ERPNext / Frappe Framework from Claude Code: bundled skills, slash commands, specialized agents, and a live MCP connection to your ERPNext instance.

## What's inside

| Component | Items |
|---|---|
| **Skills** | `erpnext` (API, DocTypes, reports, workflows, troubleshooting), `frappe-frontend` (Vue 3 + frappe-ui + Tailwind portals/PWAs), `frappe-ui` (component library, theming, composables), `frappe-insights` (BI queries, charts, dashboards), `frappe-code-review` (Frappe-specific code & security review) |
| **Commands** | `/erp-query` (read data), `/erp-doctype` (design DocTypes), `/erp-report` (build reports), `/erp-fix` (troubleshoot errors) |
| **Agents** | `erpnext-developer` (code tasks), `erpnext-analyst` (business data analysis, read-only) |
| **MCP** | Live connection to your ERPNext instance via `@casys/mcp-erpnext` (community server, 120+ tools) |

## Setup

### 1. MCP credentials (environment variables)

The MCP server needs three environment variables. Set them in your shell profile (`~/.bashrc` / `~/.zshrc`):

```bash
export ERPNEXT_URL="https://your-site.erpnext.com"   # your ERPNext site URL
export ERPNEXT_API_KEY="xxxx"
export ERPNEXT_API_SECRET="xxxx"
```

To generate an API key/secret: in ERPNext, go to **User → API Access → Generate Keys**. It is safest to create a dedicated user and give it only the roles it needs.

Requires Node.js ≥ 20 (the MCP server runs via `npx`).

### 2. Install the plugin

**Option A — local testing (quick to try):**

```bash
claude --plugin-dir /path/to/erpnext-toolkit
```

**Option B — via a marketplace (to share with a team):**

1. Push this folder to a GitHub repo and add a `.claude-plugin/marketplace.json` at the repo root.
2. In Claude Code:

```
/plugin marketplace add your-github-user/your-repo
/plugin install erpnext-toolkit
```

### 3. Verify

- Run `/erp-query outstanding sales invoices` to check that the MCP connection works
- `/agents` — you should see the two agents listed
- The skills trigger automatically whenever ERPNext/Frappe comes up

## Security notes

- The MCP server can both read and write, according to your API key's permissions — use a least-privilege user on production instances.
- The `erpnext-analyst` agent is read-only by its rules, but the final control is the API user's roles.
- `@casys/mcp-erpnext` is a community project — review it before installing: https://github.com/Casys-AI/mcp-erpnext

## Docs

- Claude Code plugins: https://code.claude.com/docs/en/plugins-reference
- ERPNext: https://docs.frappe.io/erpnext
