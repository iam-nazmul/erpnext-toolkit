# PWA (installable, offline-capable Frappe frontends)

Frappe's own mobile experiences (e.g., HRMS mobile) are PWAs: the Vue SPA plus a web app manifest and a service worker via **vite-plugin-pwa**. Users "install" from the browser; it launches full-screen like a native app.

## Setup

```bash
npm install -D vite-plugin-pwa
```

```js
// vite.config.js
import { VitePWA } from 'vite-plugin-pwa'

export default {
  plugins: [
    frappeui(), vue(),
    VitePWA({
      registerType: 'autoUpdate',           // new SW activates on next load — right default for business apps
      strategies: 'generateSW',
      manifest: {
        name: 'Acme Field App',
        short_name: 'Field',
        start_url: '/frontend',             // must match the served route
        scope: '/frontend',
        display: 'standalone',
        theme_color: '#0f0f0f',
        background_color: '#ffffff',
        icons: [
          { src: '/assets/my_app/frontend/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: '/assets/my_app/frontend/icon-512.png', sizes: '512x512', type: 'image/png' },
          { src: '/assets/my_app/frontend/icon-512-maskable.png', sizes: '512x512',
            type: 'image/png', purpose: 'maskable' },
        ],
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        navigateFallback: '/frontend',
        navigateFallbackDenylist: [/^\/api/, /^\/app/, /^\/files/, /^\/assets(?!\/my_app\/frontend)/],
        runtimeCaching: [
          {
            urlPattern: /^\/api\/method\/my_app\.api\.get_masters/,
            handler: 'StaleWhileRevalidate',
            options: { cacheName: 'masters', expiration: { maxAgeSeconds: 3600 } },
          },
        ],
      },
    }),
  ],
}
```

Critical details:

- **Scope/start_url must match the Frappe serving route** (`/frontend`, `/hrms`, …). A `/` scope will fight with Desk and website pages.
- **Never let the service worker cache `/api`** with cache-first strategies — business data must not go stale silently. `NetworkFirst`/`StaleWhileRevalidate` only for genuinely cacheable reads (masters, settings), never for transactional lists, and exclude `/api` from `navigateFallback`.
- Icons live in `public/` of the frontend and end up under the app's asset route — reference them by their **served** path.
- HTTPS is required for service workers (localhost is exempt for dev).

## Offline strategy — be honest about it

A Frappe PWA is online-first. Realistic offline tiers:

1. **Shell offline (default, cheap)**: app loads offline, shows cached masters, tells the user data actions need connectivity. This is what the config above gives you.
2. **Read cache**: runtime-cache selected GET endpoints; show a "last synced" indicator (`navigator.onLine` + a timestamp you store on successful fetches).
3. **Offline writes (expensive — only if truly required)**: queue mutations in IndexedDB (Workbox BackgroundSync or hand-rolled), replay on reconnect, with an idempotency key (client-generated UUID in a custom field) so retries don't duplicate documents, and a server-side conflict policy. Frappe's `modified`-timestamp check will reject stale updates — surface those as conflicts to the user, don't auto-overwrite.

Recommend tier 1 or 2 unless the user explicitly has field-workers-without-signal requirements; tier 3 is a project, not a config flag.

## Update flow

`registerType: 'autoUpdate'` silently updates on next navigation. If the team wants a "New version available — Reload" toast instead, use `registerType: 'prompt'` + `useRegisterSW()` from `virtual:pwa-register/vue` and wire it to a frappe-ui `toast`. During development, unregister rogue service workers (devtools → Application → Service Workers) when things get weird — a stale SW is the classic "works in incognito" bug.

## Native-ish touches

- `display: standalone` removes browser chrome; add your own back button UX (router history) since there's no URL bar.
- iOS: add `apple-touch-icon` link tags; iOS ignores parts of the manifest and has its own splash behavior — test on a real device.
- Push notifications: possible via Web Push, but Frappe-side you'll be building the subscription storage + a send hook (`frappe.enqueue` on doc events) yourself; HRMS-style apps often skip push and rely on email/in-app.
- App shortcuts (`manifest.shortcuts`) give long-press quick actions ("New Expense", "Check In") that deep-link into routes.

## Install & audit checklist

- Lighthouse → PWA audit passes (manifest, SW, icons, HTTPS).
- Install prompt appears on Android/desktop Chrome (`beforeinstallprompt` can be captured to show your own "Install app" button).
- Deep link to a sub-route while offline loads the shell (navigateFallback working).
- Login state: the session cookie survives installation (same origin), but test the logged-out flow inside the installed app — your login page must work within the SPA, not bounce to Desk's `/login` (or if it does, ensure redirect back works).
