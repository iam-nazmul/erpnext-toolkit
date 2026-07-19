# Privacy Policy — ERPNext Toolkit

**Effective date:** 19 July 2026
**Applies to:** the `erpnext-toolkit` Claude Code plugin ("the Plugin"), published at https://github.com/iam-nazmul/erpnext-toolkit

## Summary

The Plugin does not collect, store, transmit, or sell any of your data to the Plugin author. It runs entirely on your machine and connects only to services **you** configure: your own ERPNext instance and Anthropic's Claude (via Claude Code, which you already use). We (the Plugin author) never see your credentials, your ERP data, or your conversations.

## 1. What the Plugin is

ERPNext Toolkit is an open-source plugin for Claude Code. It bundles skills, slash commands, and agents (static instruction files), and configures a local MCP server (`@casys/mcp-erpnext`, a community project) that talks to the ERPNext instance you specify.

## 2. Data you provide

At install time, Claude Code prompts you for:

- **ERPNext URL** — the address of your ERPNext instance
- **ERPNext API key and API secret** — credentials for an ERPNext user you control

These values are stored **locally on your machine** by Claude Code as plugin user configuration. They are:

- never transmitted to the Plugin author;
- never transmitted to any analytics, telemetry, or third-party service by the Plugin;
- used only to authenticate requests from the local MCP server to your ERPNext instance.

You can delete these credentials at any time by uninstalling the plugin or removing the plugin user config, and revoke them at any time from within ERPNext (**User → API Access**).

## 3. How your data flows when you use the Plugin

When you ask Claude to read or write ERP data, the following happens:

1. **Your machine → your ERPNext instance.** The local MCP server sends API requests (authenticated with your key/secret) directly to the ERPNext URL you configured. No intermediary server is involved.
2. **Your machine → Anthropic.** As with all Claude Code usage, your prompts and any tool results (which may include ERP data returned by your instance, such as customer names, invoices, stock levels, or employee records) are sent to Anthropic to generate responses. This flow is governed by Anthropic's terms and privacy policy, not by the Plugin: https://www.anthropic.com/legal/privacy

The Plugin itself adds **no other network destinations**. It contains no telemetry, analytics, tracking, or "phone home" functionality.

## 4. Data the Plugin author collects

**None.** We operate no servers, receive no usage data, no crash reports, no credentials, and no ERP data. The Plugin is distributed as source code via GitHub; GitHub's own privacy policy applies to your interactions with the repository (stars, issues, clones).

## 5. Third-party components

| Component | Role | Governed by |
|---|---|---|
| **Claude Code / Anthropic** | Runs the Plugin; processes your prompts and tool results | [Anthropic Privacy Policy](https://www.anthropic.com/legal/privacy) |
| **`@casys/mcp-erpnext`** | Community MCP server, fetched via `npx` from the npm registry and run locally | [Its repository](https://github.com/Casys-AI/mcp-erpnext) and the npm registry's terms |
| **Your ERPNext instance** | The system of record for your business data | You (or your ERPNext hosting provider) |

We do not control these components. Review them independently — in particular, `@casys/mcp-erpnext` is a community project we recommend auditing before use on production systems.

## 6. Sensitive business data — your responsibilities

Because the MCP connection can read (and, with sufficient permissions, write) live ERP data, we strongly recommend:

- Using a **dedicated, least-privilege ERPNext user** for the API key, granting only the roles needed.
- Not connecting production instances containing regulated data (e.g. personal data subject to GDPR, health, or payroll data) unless your organization's policies permit sending such data to Anthropic for processing.
- Reviewing Claude Code's data settings if your organization requires zero data retention or has an Anthropic enterprise agreement.

You are the data controller for the ERP data you expose through this Plugin; the Plugin author processes nothing on your behalf.

## 7. Children's privacy

The Plugin is a developer tool not directed at children and does not knowingly collect data from anyone, including children.

## 8. Changes to this policy

Changes will be published in this file in the GitHub repository, with the effective date updated above. Material changes will be noted in release notes.

## 9. Contact

Questions or concerns: **MD. NAZMUL HOSSAIN** — nazmul.cse48@gmail.com, or open an issue at https://github.com/iam-nazmul/erpnext-toolkit/issues
