---
name: frappe-frontend
description: Build custom frontends, portals, and PWAs for ERPNext/Frappe using Vue 3, frappe-ui, and TailwindCSS. Use this skill whenever the user wants a custom UI on top of Frappe/ERPNext — a customer portal, mobile PWA, dashboard, kiosk, or SPA — or mentions frappe-ui, doppio, "bench add-frappe-ui", Vue components for Frappe, createResource/createListResource, or Tailwind styling in a Frappe app. Also use for questions about serving a Vite/Vue build from a Frappe site, SPA routing via website_route_rules, CSRF tokens in custom frontends, or making a Frappe app installable/offline-capable as a PWA.
---

# Frappe Frontend (Vue 3 + frappe-ui + Tailwind + PWA)

The modern Frappe frontend stack — used by Frappe's own products (Helpdesk, CRM, Insights, HRMS mobile) — is:

**Vue 3 (Composition API) + Vite + TailwindCSS + frappe-ui**, served either standalone (dev, proxied to the site) or built into a Frappe app's `www/` folder (production), optionally wrapped as a **PWA**.

This is distinct from Desk (the built-in admin UI) and from classic Jinja portal pages. Reach for this stack when the user needs a tailored experience for customers, field staff, or mobile — not when a Desk customization would do (see the `erpnext` skill's low-code reference for that).

## How to use this skill

| Task | Reference file |
|---|---|
| Project scaffolding, dev proxy, building into a Frappe app, deployment | `references/project-setup.md` |
| Data fetching (createResource / createListResource / createDocumentResource), auth/session, realtime | `references/frappe-ui-data.md` |
| frappe-ui components + Tailwind conventions | `references/components-and-tailwind.md` |
| Vue 3 patterns as used in this stack (composition API, router, composables) | `references/vue-patterns.md` |
| Making it a PWA (manifest, service worker, offline, install) | `references/pwa.md` |

## Architecture in one diagram

```
Browser (Vue SPA, built by Vite)
   │  /api/method/..., /api/resource/...   ← frappe-ui resources wrap these
   ▼
Frappe site (nginx → gunicorn)
   ├── serves built SPA from app/www/<name>/  (production)
   └── socket.io for realtime               (optional)

Dev mode: Vite dev server (:8080) proxies /api, /app, /files → Frappe (:8000)
```

Key implications:
- The SPA authenticates with the **same session cookie** as the site — a logged-in ERPNext user is logged into your portal. Permissions are the server's job; the SPA is untrusted.
- All data access is the standard Frappe API (see the `erpnext` skill's `rest-api.md`); frappe-ui just gives you reactive wrappers around it.
- For portal users, create Website Users with appropriate roles/User Permissions; never widen DocType permissions just to make the SPA work — add whitelisted API methods that enforce their own checks instead.

## Ground rules

- Always start from a working scaffold (`bench add-frappe-ui` via the doppio bench extension, or the `frappe-ui-starter` template) rather than wiring Vite/Tailwind/proxy by hand.
- Match versions: frappe-ui moves fast; check the installed version's docs/source (`node_modules/frappe-ui`) before asserting component props. When in doubt, read the component source — it's small and readable.
- Use frappe-ui's Tailwind preset and design tokens instead of hardcoding colors; it keeps the UI consistent with Frappe's look and dark-mode ready.
- CSRF: production builds served from the site must set `window.csrf_token` (the boilerplate's `index.html` ↔ `www/*.html` handles this); if POSTs fail with CSRFTokenError only in production, this is why.
