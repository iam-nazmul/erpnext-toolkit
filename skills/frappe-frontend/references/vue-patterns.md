# Vue 3 Patterns for Frappe Frontends

Frappe frontends use **Vue 3 with `<script setup>` (Composition API)**. This file covers the subset of Vue that these apps actually use, plus stack-specific conventions.

## Component anatomy

```vue
<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const props = defineProps({ ticketId: { type: String, required: true } })
const emit = defineEmits(['updated'])

const search = ref('')
const filtered = computed(() => items.value.filter(i => i.subject.includes(search.value)))

watch(() => props.ticketId, (id) => { /* refetch */ })
onMounted(() => { /* dom-dependent setup */ })
</script>

<template>
  <input v-model="search" />
  <div v-for="item in filtered" :key="item.name">{{ item.subject }}</div>
</template>
```

Essentials: `ref()` for primitives (access with `.value` in script, bare in template), `reactive()` for objects, `computed()` for derived state, `watch`/`watchEffect` for side effects, `v-model` for two-way binding, `:prop` / `@event` binding, `v-if` vs `v-show`, `:key` on every `v-for`.

## Where state lives in this stack

Most "state management" is unnecessary — **frappe-ui resources are the store**:

- Server data → `createResource`/`createListResource`/`createDocumentResource`, shared across components via the `cache` key.
- Session → the reactive `session` module (see `frappe-ui-data.md`).
- Truly client-only shared state (UI toggles, current view) → a plain composable exporting `ref`s, or Pinia if the app is large. Frappe products mostly use composables + resources, not Vuex/Pinia.

Composable pattern:

```js
// src/composables/useSidebar.js
import { ref } from 'vue'
const isOpen = ref(true)                       // module-level = shared singleton
export function useSidebar() {
  return { isOpen, toggle: () => (isOpen.value = !isOpen.value) }
}
```

## Router

```js
import { createRouter, createWebHistory } from 'vue-router'
import { session } from '@/data/session'

const router = createRouter({
  history: createWebHistory('/frontend'),      // must match the www route the SPA is served on
  routes: [
    { path: '/', name: 'Home', component: () => import('@/pages/Home.vue') },
    { path: '/ticket/:ticketId', name: 'Ticket',
      component: () => import('@/pages/Ticket.vue'), props: true },
    { path: '/login', name: 'Login', component: () => import('@/pages/Login.vue') },
  ],
})

router.beforeEach((to) => {
  if (to.name !== 'Login' && !session.isLoggedIn) {
    return { name: 'Login', query: { redirect: to.fullPath } }
  }
})
export default router
```

Notes: the `history` base must match the serving route (`/frontend`, `/crm`, …) or deep links 404 in production even though dev works. Lazy-load pages with dynamic imports. Pass route params to components via `props: true`.

## Patterns that recur in Frappe product code

- **Resource-per-page**: create the page's resources in `<script setup>` keyed off route props; `watch` the prop to `reload()` on navigation between records.
- **Optimistic UI**: `resource.setData()` immediately, then `setValue.submit()`; on error, `reload()`.
- **Debounced search**: watch a search `ref` with a debounce (lodash-es `debounce` or a tiny composable) → update list resource filters → `reload()`.
- **Empty/loading/error triad**: every data view renders three branches — skeleton or `<Spinner>` while `resource.loading`, `<ErrorMessage>` on `resource.error`, empty-state when `data.length === 0`.
- **Feature layout**: `src/pages/` (routed views), `src/components/` (shared), `src/data/` (resources/session), `src/composables/`, `src/utils/`.
- **Dayjs for dates** (`dayjs(doc.modified).fromNow()`), matching Frappe's date strings.

## Common pitfalls

- Losing reactivity by destructuring: `const { data } = tickets` breaks; use `tickets.data` in templates or `toRefs`.
- Forgetting `.value` in `<script>` (or adding it in templates).
- Mutating a resource's `doc`/`data` and expecting persistence — persistence goes through `setValue`/`insert`/`save` submits.
- `v-for` without stable `:key` (use `name`) causing UI glitches on reload.
- Doing permission logic client-side only — hide UI by role for UX, but enforce on the server.
