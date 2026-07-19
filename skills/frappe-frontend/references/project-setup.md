# Project Setup & Deployment

## Scaffolding

**Option A — doppio bench extension (recommended inside a bench):**

```bash
bench get-app https://github.com/NagariaHussain/doppio   # install the extension once
bench add-frappe-ui                                      # scaffold frontend/ inside your custom app
# older/simpler SPA scaffold (Vue or React, without frappe-ui):
bench add-spa --app my_app
```

This creates `apps/my_app/frontend/` with Vue 3 + Vite + Tailwind + frappe-ui preconfigured, plus the hooks and build wiring described below.

**Option B — standalone starter (no bench needed for the frontend):**

```bash
npx degit frappe/frappe-ui-starter frontend
cd frontend && npm install && npm run dev
```

Point its proxy at any reachable Frappe site.

## Dev server & proxy

Vite runs on its own port and proxies Frappe paths to the site so cookies/API work:

```js
// vite.config.js
import frappeui from 'frappe-ui/vite'
import vue from '@vitejs/plugin-vue'

export default {
  plugins: [frappeui(), vue()],       // frappeui() sets up the proxy + build defaults
  build: {
    outDir: '../my_app/public/frontend',
    emptyOutDir: true,
    target: 'es2015',
  },
}
```

The proxy reads the bench's `common_site_config.json` (webserver port) when scaffolded via doppio; the standalone starter has an explicit proxy map for `/api`, `/assets`, `/files`. During dev, log into the Frappe site once in the same browser — the session cookie is shared through the proxy.

## Serving the built SPA from Frappe

Production pattern (what doppio wires up):

1. `npm run build` outputs into the app, e.g. `my_app/public/frontend` and copies `index.html` to `my_app/www/frontend.html`.
2. SPA routing — all sub-paths must serve that HTML:

```python
# hooks.py
website_route_rules = [
    {"from_route": "/frontend/<path:app_path>", "to_route": "frontend"},
]
```

3. CSRF token injection — the built `frontend.html` includes:

```html
<script>window.csrf_token = '{{ frappe.session.csrf_token }}';</script>
```

frappe-ui's request helper sends it as `X-Frappe-CSRF-Token` on writes. Missing token = CSRFTokenError on POST in production only.

4. Access at `https://site/frontend`. Rename routes by changing the www filename + route rule (products use `/helpdesk`, `/crm`, `/hrms`...).

Redirect logged-out users either in a router guard (check session, redirect to `/login?redirect-to=...`) or by making the www page require login.

## Build/deploy checklist

- `bench build` does **not** build your Vite app by default; add it to your app's build script or CI (`cd frontend && npm ci && npm run build`), commit or artifact the output per your deploy strategy.
- Set `base` in Vite if assets 404 (assets are served from `/assets/my_app/frontend/`).
- Test a production build locally with `npm run build && bench restart` before shipping — dev-proxy-only bugs (CSRF, absolute URLs, socket.io path) surface here.
- Multi-tenant benches: the SPA is per-app, served on every site with the app installed; keep site-specific config server-side, not baked into the bundle.
