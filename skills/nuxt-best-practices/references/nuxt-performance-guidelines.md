# Nuxt Best Practices

**Version 1.0.0**
Nuxt 3 Performance Guidelines
January 2026

> **Note:**
> This document is mainly for agents and LLMs to follow when maintaining,
> generating, or refactoring Nuxt 3 and Vue 3 codebases. Humans
> may also find it useful, but guidance here is optimized for automation
> and consistency by AI-assisted workflows.

---

## Abstract

Comprehensive performance optimization guide for Nuxt 3 and Vue 3 applications, designed for AI agents and LLMs. Contains 40+ rules across 8 categories, prioritized by impact from critical (eliminating waterfalls, reducing bundle size) to incremental (advanced patterns). Each rule includes detailed explanations, real-world examples comparing incorrect vs. correct implementations, and specific impact metrics to guide automated refactoring and code generation.

---

## Table of Contents

1. [Eliminating Waterfalls](#1-eliminating-waterfalls) — **CRITICAL**
   - 1.1 [Defer Await Until Needed](#11)
   - 1.2 [Parallel Data Fetching with Promise.all()](#12)
   - 1.3 [Use Multiple useFetch/useAsyncData Calls](#13)
   - 1.4 [Prevent Waterfall Chains in Server Routes](#14)
   - 1.5 [Strategic Suspense with Lazy Components](#15)
2. [Bundle Size Optimization](#2-bundle-size-optimization) — **CRITICAL**
   - 2.1 [Leverage Auto-Imports](#21)
   - 2.2 [Lazy Load Heavy Components](#22)
   - 2.3 [Defer Non-Critical Third-Party Libraries](#23)
   - 2.4 [Optimize Icon Imports](#24)
   - 2.5 [Tree-Shake Unused Nuxt Modules](#25)
3. [Server-Side Performance](#3-server-side-performance) — **HIGH**
   - 3.1 [Use Cached Composables](#31)
   - 3.2 [Minimize Hydration Payload](#32)
   - 3.3 [Use Nitro Route Rules](#33)
   - 3.4 [Parallelize Server-Side Data Fetching](#34)
   - 3.5 [Use Nitro Storage for Caching](#35)
4. [Client-Side Data Fetching](#4-client-side-data-fetching) — **MEDIUM-HIGH**
   - 4.1 [Use useFetch for API Calls](#41)
   - 4.2 [Deduplicate Requests with Keys](#42)
   - 4.3 [Use Lazy Fetch for Non-Critical Data](#43)
   - 4.4 [Refresh Data with refreshNuxtData](#44)
5. [Reactivity Optimization](#5-reactivity-optimization) — **MEDIUM**
   - 5.1 [Use Computed Instead of Methods](#51)
   - 5.2 [Avoid Unnecessary Watchers](#52)
   - 5.3 [Use Shallow Refs for Large Objects](#53)
   - 5.4 [Narrow Watch Dependencies](#54)
   - 5.5 [Use Trigger Refs for Manual Updates](#55)
   - 5.6 [Defer Reactive State Reads](#56)
6. [Rendering Performance](#6-rendering-performance) — **MEDIUM**
   - 6.1 [Animate Wrapper Divs, Not SVG Elements](#61)
   - 6.2 [Use Virtual Scrolling for Long Lists](#62)
   - 6.3 [Use ClientOnly for Browser-Specific Code](#63)
   - 6.4 [Use v-once for Static Content](#64)
   - 6.5 [Use v-memo for Expensive Lists](#65)
   - 6.6 [Use KeepAlive for Component Caching](#66)
   - 6.7 [Use v-show vs v-if Appropriately](#67)
7. [JavaScript Performance](#7-javascript-performance) — **LOW-MEDIUM**
   - 7.1 [Batch DOM Changes](#71)
   - 7.2 [Build Index Maps for Repeated Lookups](#72)
   - 7.3 [Cache Property Access in Loops](#73)
   - 7.4 [Cache Repeated Function Calls](#74)
   - 7.5 [Use toSorted() for Immutability](#75)
   - 7.6 [Combine Multiple Array Iterations](#76)
   - 7.7 [Early Return from Functions](#77)
   - 7.8 [Use Set/Map for O(1) Lookups](#78)
8. [Advanced Patterns](#8-advanced-patterns) — **LOW**
   - 8.1 [Use effectScope for Grouped Effects](#81)
   - 8.2 [Custom Refs for External State](#82)
   - 8.3 [Use Render Functions for Dynamic Components](#83)

---

## 1. Eliminating Waterfalls

**Impact: CRITICAL**

Waterfalls are the #1 performance killer. Each sequential await adds full network latency. Eliminating them yields the largest gains.

### 1.1 Defer Await Until Needed

Move `await` operations into the branches where they're actually used to avoid blocking code paths that don't need them.

**Incorrect: blocks both branches**

```typescript
// composables/useUser.ts
export const useUserData = async (userId: string, skipProcessing: boolean) => {
  const userData = await $fetch(`/api/users/${userId}`)

  if (skipProcessing) {
    // Returns immediately but still waited for userData
    return { skipped: true }
  }

  // Only this branch uses userData
  return processUserData(userData)
}
```

**Correct: only blocks when needed**

```typescript
// composables/useUser.ts
export const useUserData = async (userId: string, skipProcessing: boolean) => {
  if (skipProcessing) {
    // Returns immediately without waiting
    return { skipped: true }
  }

  // Fetch only when needed
  const userData = await $fetch(`/api/users/${userId}`)
  return processUserData(userData)
}
```

**Another example: early return in server route**

```typescript
// server/api/resource/[id].put.ts
// Incorrect: always fetches permissions
export default defineEventHandler(async (event) => {
  const permissions = await fetchPermissions(event)
  const resource = await getResource(event.context.params.id)

  if (!resource) {
    return { error: 'Not found' }
  }

  if (!permissions.canEdit) {
    return { error: 'Forbidden' }
  }

  return await updateResource(resource, permissions)
})

// Correct: fetches only when needed
export default defineEventHandler(async (event) => {
  const resource = await getResource(event.context.params.id)

  if (!resource) {
    return { error: 'Not found' }
  }

  const permissions = await fetchPermissions(event)

  if (!permissions.canEdit) {
    return { error: 'Forbidden' }
  }

  return await updateResource(resource, permissions)
})
```

### 1.2 Parallel Data Fetching with Promise.all()

When async operations have no interdependencies, execute them concurrently using `Promise.all()`.

**Incorrect: sequential execution, 3 round trips**

```typescript
// pages/dashboard.vue
const user = await $fetch('/api/user')
const posts = await $fetch('/api/posts')
const comments = await $fetch('/api/comments')
```

**Correct: parallel execution, 1 round trip**

```typescript
// pages/dashboard.vue
const [user, posts, comments] = await Promise.all([
  $fetch('/api/user'),
  $fetch('/api/posts'),
  $fetch('/api/comments')
])
```

**In composables:**

```typescript
// composables/useDashboard.ts
export const useDashboard = () => {
  const { data: user } = useFetch('/api/user')
  const { data: posts } = useFetch('/api/posts')
  const { data: comments } = useFetch('/api/comments')

  return { user, posts, comments }
}
```

All three fetches execute in parallel automatically because `useFetch` is non-blocking.

### 1.3 Use Multiple useFetch/useAsyncData Calls

Nuxt's composables execute in parallel when called in the same context. Use multiple calls instead of one sequential fetch.

**Incorrect: sequential fetching**

```typescript
// pages/profile.vue
<script setup>
const { data } = await useAsyncData('profile', async () => {
  const user = await $fetch('/api/user')
  const posts = await $fetch('/api/posts', {
    query: { userId: user.id }
  })
  return { user, posts }
})
</script>
```

**Correct: parallel fetching**

```typescript
// pages/profile.vue
<script setup>
const { data: user } = await useFetch('/api/user')
const { data: posts } = await useFetch('/api/posts', {
  query: { userId: user.value?.id }
})
</script>
```

Both fetches start immediately. The second fetch reactively updates when user data arrives.

### 1.4 Prevent Waterfall Chains in Server Routes

In server routes and API endpoints, start independent operations immediately.

**Incorrect: config waits for auth**

```typescript
// server/api/data.get.ts
export default defineEventHandler(async (event) => {
  const session = await getSession(event)
  const config = await fetchConfig()
  const data = await fetchData(session.user.id)
  return { data, config }
})
```

**Correct: auth and config start immediately**

```typescript
// server/api/data.get.ts
export default defineEventHandler(async (event) => {
  const sessionPromise = getSession(event)
  const configPromise = fetchConfig()

  const session = await sessionPromise
  const [config, data] = await Promise.all([
    configPromise,
    fetchData(session.user.id)
  ])

  return { data, config }
})
```

### 1.5 Strategic Suspense with Lazy Components

Use lazy components with Suspense to show the wrapper UI faster while data loads.

**Incorrect: page blocked by data fetching**

```vue
<!-- pages/dashboard.vue -->
<script setup>
const { data } = await useFetch('/api/dashboard')
</script>

<template>
  <div>
    <AppHeader />
    <AppSidebar />
    <DashboardContent :data="data" />
    <AppFooter />
  </div>
</template>
```

The entire layout waits for data even though only the content section needs it.

**Correct: layout shows immediately, content streams in**

```vue
<!-- pages/dashboard.vue -->
<template>
  <div>
    <AppHeader />
    <AppSidebar />
    <Suspense>
      <DashboardContent />
      <template #fallback>
        <DashboardSkeleton />
      </template>
    </Suspense>
    <AppFooter />
  </div>
</template>
```

```vue
<!-- components/DashboardContent.vue -->
<script setup>
const { data } = await useFetch('/api/dashboard')
</script>

<template>
  <div>{{ data }}</div>
</template>
```

Header, Sidebar, and Footer render immediately. Only DashboardContent waits for data.

---

## 2. Bundle Size Optimization

**Impact: CRITICAL**

Reducing initial bundle size improves Time to Interactive and Largest Contentful Paint.

### 2.1 Leverage Auto-Imports

Nuxt 3 auto-imports components, composables, and utilities. Use this instead of manual imports to enable better tree-shaking.

**Incorrect: manual imports**

```vue
<script setup>
import { ref, computed, watch } from 'vue'
import { useFetch } from '#app'
import MyComponent from '~/components/MyComponent.vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>
```

**Correct: auto-imports**

```vue
<script setup>
const count = ref(0)
const doubled = computed(() => count.value * 2)
const { data } = useFetch('/api/data')
</script>

<template>
  <MyComponent />
</template>
```

Auto-imports enable better tree-shaking and reduce bundle size by eliminating unused imports.

### 2.2 Lazy Load Heavy Components

Use `defineAsyncComponent` or lazy component syntax for large components not needed on initial render.

**Incorrect: Monaco bundles with main chunk ~300KB**

```vue
<script setup>
import MonacoEditor from '~/components/MonacoEditor.vue'
</script>

<template>
  <MonacoEditor v-if="showEditor" />
</template>
```

**Correct: Monaco loads on demand**

```vue
<script setup>
const MonacoEditor = defineAsyncComponent(() =>
  import('~/components/MonacoEditor.vue')
)
</script>

<template>
  <MonacoEditor v-if="showEditor" />
</template>
```

**Alternative: Lazy prefix (auto-imports)**

```vue
<template>
  <LazyMonacoEditor v-if="showEditor" />
</template>
```

Any component prefixed with `Lazy` is automatically code-split.

### 2.3 Defer Non-Critical Third-Party Libraries

Analytics, logging, and error tracking don't block user interaction. Load them after hydration.

**Incorrect: blocks initial bundle**

```vue
<!-- app.vue -->
<script setup>
import { initAnalytics } from '~/lib/analytics'

onMounted(() => {
  initAnalytics()
})
</script>
```

**Correct: loads after hydration**

```vue
<!-- plugins/analytics.client.ts -->
export default defineNuxtPlugin(() => {
  // Runs only on client after hydration
  import('~/lib/analytics').then(({ initAnalytics }) => {
    initAnalytics()
  })
})
```

The `.client.ts` suffix ensures it only runs client-side, and dynamic import defers loading.

### 2.4 Optimize Icon Imports

Icon libraries can be massive. Import individual icons instead of the entire library.

**Incorrect: imports entire library**

```vue
<script setup>
import { Icon } from '@iconify/vue'
</script>

<template>
  <Icon icon="mdi:check" />
  <Icon icon="mdi:close" />
</template>
```

**Correct: use Nuxt Icon module with auto-imports**

```bash
npm install --save-dev @nuxt/icon
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/icon']
})
```

```vue
<template>
  <Icon name="mdi:check" />
  <Icon name="mdi:close" />
</template>
```

Icons are loaded on-demand and cached.

**For custom icon libraries:**

```vue
<script setup>
// Import only needed icons
import IconCheck from '~/assets/icons/check.svg'
import IconClose from '~/assets/icons/close.svg'
</script>
```

### 2.5 Tree-Shake Unused Nuxt Modules

Remove unused Nuxt modules and auto-imports to reduce bundle size.

**Check what's imported:**

```bash
npx nuxi analyze
```

**Disable unused auto-imports:**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  imports: {
    // Disable auto-imports if not using them
    autoImport: false,

    // Or selectively disable specific imports
    presets: [
      {
        from: 'vue',
        imports: ['ref', 'computed', 'watch']
        // Only import what you use
      }
    ]
  }
})
```

---

## 3. Server-Side Performance

**Impact: HIGH**

Optimizing server-side rendering and data fetching eliminates server-side waterfalls and reduces response times.

### 3.1 Use Cached Composables

Cache expensive server-side operations using `defineCachedFunction` or `cachedFunction`.

**Incorrect: fetches on every request**

```typescript
// server/api/user.get.ts
export default defineEventHandler(async (event) => {
  const userId = getQuery(event).id
  const user = await db.user.findUnique({ where: { id: userId } })
  return user
})
```

**Correct: cached per request**

```typescript
// server/utils/cache.ts
export const getCachedUser = defineCachedFunction(async (userId: string) => {
  return await db.user.findUnique({ where: { id: userId } })
}, {
  maxAge: 60 * 5, // 5 minutes
  name: 'user',
  getKey: (userId) => userId
})

// server/api/user.get.ts
export default defineEventHandler(async (event) => {
  const userId = getQuery(event).id as string
  return await getCachedUser(userId)
})
```

### 3.2 Minimize Hydration Payload

The payload sent from server to client during hydration can be large. Minimize it by only including necessary data.

**Incorrect: sends all 50 fields**

```vue
<!-- pages/profile.vue -->
<script setup>
const { data: user } = await useFetch('/api/user')
</script>

<template>
  <div>{{ user.name }}</div>
</template>
```

If the API returns 50 fields but you only need `name`, all fields are still serialized and sent to the client.

**Correct: only fetch what you need**

```vue
<!-- pages/profile.vue -->
<script setup>
const { data: user } = await useFetch('/api/user', {
  transform: (data) => ({ name: data.name })
})
</script>

<template>
  <div>{{ user.name }}</div>
</template>
```

Or create a dedicated endpoint:

```typescript
// server/api/user/name.get.ts
export default defineEventHandler(async (event) => {
  const user = await db.user.findUnique({
    where: { id: event.context.userId },
    select: { name: true }
  })
  return user
})
```

### 3.3 Use Nitro Route Rules

Configure route behavior using route rules for static generation, caching, and redirects.

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // Static generation
    '/about': { prerender: true },

    // SWR caching (stale-while-revalidate)
    '/api/posts': { swr: 60 }, // Cache for 60 seconds

    // ISR (Incremental Static Regeneration)
    '/blog/**': { isr: 3600 }, // Regenerate every hour

    // Client-side only
    '/dashboard/**': { ssr: false },

    // Redirect
    '/old-path': { redirect: '/new-path' }
  }
})
```

### 3.4 Parallelize Server-Side Data Fetching

Use component composition to parallelize server-side data fetching.

**Incorrect: Header waits for Page's fetch**

```vue
<!-- pages/index.vue -->
<script setup>
const { data: header } = await useFetch('/api/header')
</script>

<template>
  <div>
    <div>{{ header }}</div>
    <Sidebar />
  </div>
</template>

<!-- components/Sidebar.vue -->
<script setup>
const { data: items } = await useFetch('/api/sidebar')
</script>
```

Both fetches execute sequentially because the component tree renders top-down.

**Correct: both fetch simultaneously**

```vue
<!-- components/Header.vue -->
<script setup>
const { data: header } = await useFetch('/api/header')
</script>

<!-- components/Sidebar.vue -->
<script setup>
const { data: items } = await useFetch('/api/sidebar')
</script>

<!-- pages/index.vue -->
<template>
  <div>
    <Header />
    <Sidebar />
  </div>
</template>
```

Now both components fetch in parallel.

### 3.5 Use Nitro Storage for Caching

Use Nitro's storage layer for persistent caching across requests.

```typescript
// server/api/expensive.get.ts
export default defineEventHandler(async (event) => {
  const storage = useStorage('cache')
  const cached = await storage.getItem('expensive-result')

  if (cached) {
    return cached
  }

  const result = await expensiveComputation()
  await storage.setItem('expensive-result', result, {
    ttl: 60 * 60 // 1 hour
  })

  return result
})
```

Configure storage drivers:

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    storage: {
      cache: {
        driver: 'redis',
        host: process.env.REDIS_HOST,
        port: process.env.REDIS_PORT
      }
    }
  }
})
```

---

## 4. Client-Side Data Fetching

**Impact: MEDIUM-HIGH**

Efficient data fetching patterns reduce redundant network requests and improve responsiveness.

### 4.1 Use useFetch for API Calls

Always use `useFetch` or `useAsyncData` instead of manual `$fetch` for better SSR support and automatic reactivity.

**Incorrect: no SSR, no reactivity**

```vue
<script setup>
const data = ref(null)

onMounted(async () => {
  data.value = await $fetch('/api/data')
})
</script>
```

**Correct: SSR + reactivity**

```vue
<script setup>
const { data } = useFetch('/api/data')
</script>
```

Benefits:
- Executes on server during SSR
- Automatically reactive
- Deduplicates requests
- Handles loading and error states

### 4.2 Deduplicate Requests with Keys

Use unique keys to deduplicate identical requests across components.

**Incorrect: multiple instances = multiple requests**

```vue
<!-- ComponentA.vue -->
<script setup>
const { data } = useFetch('/api/user')
</script>

<!-- ComponentB.vue -->
<script setup>
const { data } = useFetch('/api/user')
</script>
```

Without a key, both components make separate requests.

**Correct: multiple instances = one request**

```vue
<!-- ComponentA.vue -->
<script setup>
const { data } = useFetch('/api/user', { key: 'user' })
</script>

<!-- ComponentB.vue -->
<script setup>
const { data } = useFetch('/api/user', { key: 'user' })
</script>
```

With the same key, only one request is made and shared.

### 4.3 Use Lazy Fetch for Non-Critical Data

Use `lazy: true` to defer data fetching until after initial render.

```vue
<script setup>
// Critical data - blocks render
const { data: user } = await useFetch('/api/user')

// Non-critical data - deferred
const { data: stats, pending } = useFetch('/api/stats', {
  lazy: true
})
</script>

<template>
  <div>
    <h1>{{ user.name }}</h1>
    <div v-if="pending">Loading stats...</div>
    <div v-else>{{ stats }}</div>
  </div>
</template>
```

### 4.4 Refresh Data with refreshNuxtData

Refresh specific cached data using `refreshNuxtData`.

```vue
<script setup>
const { data } = useFetch('/api/posts', { key: 'posts' })

const refresh = () => {
  refreshNuxtData('posts')
}
</script>

<template>
  <button @click="refresh">Refresh</button>
</template>
```

For multiple keys:

```typescript
refreshNuxtData(['posts', 'comments'])
```

---

## 5. Reactivity Optimization

**Impact: MEDIUM**

Optimizing Vue's reactivity system reduces unnecessary computations and re-renders.

### 5.1 Use Computed Instead of Methods

Computed properties cache results; methods recompute every time.

**Incorrect: recalculates on every render**

```vue
<script setup>
const items = ref([/* large array */])

function filteredItems() {
  return items.value.filter(item => item.active)
}
</script>

<template>
  <div v-for="item in filteredItems()" :key="item.id">
    {{ item.name }}
  </div>
</template>
```

`filteredItems()` runs on every render, even if `items` didn't change.

**Correct: cached and reactive**

```vue
<script setup>
const items = ref([/* large array */])

const filteredItems = computed(() => {
  return items.value.filter(item => item.active)
})
</script>

<template>
  <div v-for="item in filteredItems" :key="item.id">
    {{ item.name }}
  </div>
</template>
```

Computed runs only when `items` changes.

### 5.2 Avoid Unnecessary Watchers

Don't use watchers when computed properties suffice.

**Incorrect: manual synchronization**

```vue
<script setup>
const firstName = ref('John')
const lastName = ref('Doe')
const fullName = ref('')

watch([firstName, lastName], ([first, last]) => {
  fullName.value = `${first} ${last}`
})
</script>
```

**Correct: automatic derivation**

```vue
<script setup>
const firstName = ref('John')
const lastName = ref('Doe')

const fullName = computed(() => `${firstName.value} ${lastName.value}`)
</script>
```

Use watchers only for side effects (API calls, DOM manipulation), not for derived state.

### 5.3 Use Shallow Refs for Large Objects

For large objects where you only mutate top-level properties, use `shallowRef` to skip deep reactivity.

**Incorrect: deep reactivity for large object**

```vue
<script setup>
const largeConfig = ref({
  settings: { /* 1000s of nested properties */ },
  theme: { /* deep nested object */ }
})

// Triggers deep reactivity tracking
largeConfig.value.settings.foo = 'bar'
</script>
```

**Correct: shallow reactivity**

```vue
<script setup>
const largeConfig = shallowRef({
  settings: { /* 1000s of nested properties */ },
  theme: { /* deep nested object */ }
})

// Replace entire object to trigger update
largeConfig.value = {
  ...largeConfig.value,
  settings: { ...largeConfig.value.settings, foo: 'bar' }
}
</script>
```

Use `shallowRef` when:
- Object is very large
- You only update top-level properties
- Deep reactivity isn't needed

### 5.4 Narrow Watch Dependencies

Watch specific properties instead of entire objects to minimize re-runs.

**Incorrect: re-runs on any user property change**

```vue
<script setup>
const user = ref({ id: 1, name: 'John', email: 'john@example.com', /* 50 more fields */ })

watch(user, (newUser) => {
  console.log(newUser.id)
})
</script>
```

**Correct: re-runs only when id changes**

```vue
<script setup>
const user = ref({ id: 1, name: 'John', email: 'john@example.com', /* 50 more fields */ })

watch(() => user.value.id, (newId) => {
  console.log(newId)
})
</script>
```

### 5.5 Use Trigger Refs for Manual Updates

Use `triggerRef` to manually trigger updates for shallow refs.

```vue
<script setup>
const state = shallowRef({ count: 0 })

function increment() {
  // Mutate in place
  state.value.count++
  // Manually trigger update
  triggerRef(state)
}
</script>
```

### 5.6 Defer Reactive State Reads

Don't subscribe to reactive state if you only read it inside callbacks.

**Incorrect: subscribes to all route changes**

```vue
<script setup>
const route = useRoute()

const handleShare = () => {
  const ref = route.query.ref
  shareContent({ ref })
}
</script>
```

Component re-renders on every route change, even though `ref` is only read in `handleShare`.

**Correct: reads on demand, no subscription**

```vue
<script setup>
const handleShare = () => {
  const route = useRoute()
  const ref = route.query.ref
  shareContent({ ref })
}
</script>
```

---

## 6. Rendering Performance

**Impact: MEDIUM**

Optimizing the rendering process reduces the work the browser needs to do.

### 6.1 Animate Wrapper Divs, Not SVG Elements

Many browsers don't have hardware acceleration for CSS animations on SVG elements. Wrap SVG in a `<div>` and animate the wrapper.

**Incorrect: animating SVG directly - no hardware acceleration**

```vue
<template>
  <svg class="animate-spin" width="24" height="24" viewBox="0 0 24 24">
    <circle cx="12" cy="12" r="10" stroke="currentColor" />
  </svg>
</template>

<style>
.animate-spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
</style>
```

**Correct: animating wrapper div - hardware accelerated**

```vue
<template>
  <div class="animate-spin">
    <svg width="24" height="24" viewBox="0 0 24 24">
      <circle cx="12" cy="12" r="10" stroke="currentColor" />
    </svg>
  </div>
</template>
```

### 6.2 Use Virtual Scrolling for Long Lists

For lists with hundreds or thousands of items, use virtual scrolling to render only visible items.

**Incorrect: renders all 10,000 items**

```vue
<template>
  <div class="overflow-y-auto h-screen">
    <div v-for="item in items" :key="item.id">
      {{ item.name }}
    </div>
  </div>
</template>
```

**Correct: renders only ~20 visible items**

```bash
npm install vue-virtual-scroller
```

```vue
<script setup>
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

const items = ref([/* 10,000 items */])
</script>

<template>
  <RecycleScroller
    :items="items"
    :item-size="50"
    key-field="id"
    v-slot="{ item }"
  >
    <div>{{ item.name }}</div>
  </RecycleScroller>
</template>
```

### 6.3 Use ClientOnly for Browser-Specific Code

Wrap browser-specific code with `<ClientOnly>` to prevent hydration mismatches.

**Incorrect: breaks SSR**

```vue
<script setup>
const theme = localStorage.getItem('theme') || 'light'
</script>

<template>
  <div :class="theme">Content</div>
</template>
```

**Correct: no hydration mismatch**

```vue
<script setup>
const theme = ref('light')

onMounted(() => {
  theme.value = localStorage.getItem('theme') || 'light'
})
</script>

<template>
  <ClientOnly>
    <div :class="theme">Content</div>
  </ClientOnly>
</template>
```

**Alternative: prevent flicker with placeholder**

```vue
<template>
  <ClientOnly>
    <div :class="theme">Content</div>
    <template #fallback>
      <div class="light">Content</div>
    </template>
  </ClientOnly>
</template>
```

### 6.4 Use v-once for Static Content

Use `v-once` to render content only once and skip future updates.

```vue
<template>
  <div v-once>
    <h1>{{ staticTitle }}</h1>
    <p>This content never changes</p>
  </div>
</template>
```

Useful for:
- Static headers/footers
- Terms of service
- Documentation pages

### 6.5 Use v-memo for Expensive Lists

Use `v-memo` to skip re-rendering list items when dependencies haven't changed.

**Without v-memo:**

```vue
<template>
  <div v-for="item in items" :key="item.id">
    <ExpensiveComponent :data="item" />
  </div>
</template>
```

Every item re-renders when `items` array changes.

**With v-memo:**

```vue
<template>
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id, item.updatedAt]"
  >
    <ExpensiveComponent :data="item" />
  </div>
</template>
```

Item only re-renders when `id` or `updatedAt` changes.

### 6.6 Use KeepAlive for Component Caching

Use `<KeepAlive>` to preserve component state and avoid re-rendering.

**Without KeepAlive:**

```vue
<template>
  <component :is="currentTab" />
</template>
```

Component state is lost when switching tabs.

**With KeepAlive:**

```vue
<template>
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

Component state persists when switching tabs.

**With max cache:**

```vue
<template>
  <KeepAlive :max="10">
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

### 6.7 Use v-show vs v-if Appropriately

Use `v-show` for frequently toggled content; use `v-if` for infrequent changes.

**v-show: cheaper toggle, always in DOM**

```vue
<template>
  <div v-show="isVisible">
    Content that toggles frequently
  </div>
</template>
```

**v-if: more expensive toggle, removed from DOM**

```vue
<template>
  <div v-if="isVisible">
    Heavy component that rarely changes
  </div>
</template>
```

**When to use each:**

- `v-show`: Dropdown menus, modals, tooltips (frequent toggles)
- `v-if`: Heavy components, conditional routes (infrequent changes)

---

## 7. JavaScript Performance

**Impact: LOW-MEDIUM**

Micro-optimizations for hot paths can add up to meaningful improvements.

### 7.1 Batch DOM Changes

Change multiple CSS properties via classes instead of individual style changes.

**Incorrect: multiple reflows**

```vue
<script setup>
const updateStyles = (el: HTMLElement) => {
  el.style.width = '100px'
  el.style.height = '200px'
  el.style.backgroundColor = 'blue'
  el.style.border = '1px solid black'
}
</script>
```

**Correct: single reflow**

```vue
<style>
.highlighted-box {
  width: 100px;
  height: 200px;
  background-color: blue;
  border: 1px solid black;
}
</style>

<script setup>
const updateStyles = (el: HTMLElement) => {
  el.classList.add('highlighted-box')
}
</script>
```

### 7.2 Build Index Maps for Repeated Lookups

Multiple `.find()` calls by the same key should use a Map.

**Incorrect (O(n) per lookup):**

```typescript
function processOrders(orders: Order[], users: User[]) {
  return orders.map(order => ({
    ...order,
    user: users.find(u => u.id === order.userId)
  }))
}
```

**Correct (O(1) per lookup):**

```typescript
function processOrders(orders: Order[], users: User[]) {
  const userById = new Map(users.map(u => [u.id, u]))

  return orders.map(order => ({
    ...order,
    user: userById.get(order.userId)
  }))
}
```

### 7.3 Cache Property Access in Loops

Cache object property lookups in hot paths.

**Incorrect: multiple lookups per iteration**

```typescript
for (let i = 0; i < arr.length; i++) {
  process(obj.config.settings.value)
}
```

**Correct: cached**

```typescript
const value = obj.config.settings.value
const len = arr.length
for (let i = 0; i < len; i++) {
  process(value)
}
```

### 7.4 Cache Repeated Function Calls

Use a Map to cache function results for repeated inputs.

```typescript
const cache = new Map<string, string>()

function slugify(text: string): string {
  if (cache.has(text)) {
    return cache.get(text)!
  }

  const result = text
    .toLowerCase()
    .replace(/\s+/g, '-')
    .replace(/[^\w-]/g, '')

  cache.set(text, result)
  return result
}
```

### 7.5 Use toSorted() for Immutability

`.sort()` mutates the array; use `.toSorted()` for immutability.

**Incorrect: mutates original**

```vue
<script setup>
const users = ref([/* users */])

const sorted = computed(() =>
  users.value.sort((a, b) => a.name.localeCompare(b.name))
)
</script>
```

**Correct: creates new array**

```vue
<script setup>
const users = ref([/* users */])

const sorted = computed(() =>
  users.value.toSorted((a, b) => a.name.localeCompare(b.name))
)
</script>
```

### 7.6 Combine Multiple Array Iterations

Combine multiple array iterations into a single loop.

**Incorrect: 3 iterations**

```typescript
const admins = users.filter(u => u.isAdmin)
const testers = users.filter(u => u.isTester)
const inactive = users.filter(u => !u.isActive)
```

**Correct: 1 iteration**

```typescript
const admins: User[] = []
const testers: User[] = []
const inactive: User[] = []

for (const user of users) {
  if (user.isAdmin) admins.push(user)
  if (user.isTester) testers.push(user)
  if (!user.isActive) inactive.push(user)
}
```

### 7.7 Early Return from Functions

Return early when the result is determined.

**Incorrect: processes all items**

```typescript
function validateUsers(users: User[]) {
  let hasError = false
  let errorMessage = ''

  for (const user of users) {
    if (!user.email) {
      hasError = true
      errorMessage = 'Email required'
    }
  }

  return hasError ? { valid: false, error: errorMessage } : { valid: true }
}
```

**Correct: returns immediately**

```typescript
function validateUsers(users: User[]) {
  for (const user of users) {
    if (!user.email) {
      return { valid: false, error: 'Email required' }
    }
  }

  return { valid: true }
}
```

### 7.8 Use Set/Map for O(1) Lookups

Convert arrays to Set/Map for repeated membership checks.

**Incorrect (O(n) per check):**

```typescript
const allowedIds = ['a', 'b', 'c']
items.filter(item => allowedIds.includes(item.id))
```

**Correct (O(1) per check):**

```typescript
const allowedIds = new Set(['a', 'b', 'c'])
items.filter(item => allowedIds.has(item.id))
```

---

## 8. Advanced Patterns

**Impact: LOW**

Advanced patterns for specific cases that require careful implementation.

### 8.1 Use effectScope for Grouped Effects

Use `effectScope` to group and dispose of multiple effects together.

```vue
<script setup>
let scope: EffectScope | null = null

const startMonitoring = () => {
  scope = effectScope()

  scope.run(() => {
    watch(data, () => { /* ... */ })
    watchEffect(() => { /* ... */ })
    const interval = setInterval(() => { /* ... */ }, 1000)

    onScopeDispose(() => {
      clearInterval(interval)
    })
  })
}

const stopMonitoring = () => {
  scope?.stop()
  scope = null
}
</script>
```

### 8.2 Custom Refs for External State

Use `customRef` to create refs with custom get/set logic.

```typescript
import { customRef } from 'vue'

function useDebouncedRef<T>(value: T, delay = 300) {
  let timeout: NodeJS.Timeout

  return customRef((track, trigger) => {
    return {
      get() {
        track()
        return value
      },
      set(newValue) {
        clearTimeout(timeout)
        timeout = setTimeout(() => {
          value = newValue
          trigger()
        }, delay)
      }
    }
  })
}

// Usage
const search = useDebouncedRef('', 500)
```

### 8.3 Use Render Functions for Dynamic Components

For highly dynamic components, use render functions instead of templates.

```vue
<script setup>
import { h } from 'vue'

const renderDynamic = (type: string, props: any) => {
  const componentMap = {
    button: 'button',
    link: 'a',
    text: 'span'
  }

  return h(componentMap[type] || 'div', props)
}
</script>

<template>
  <component :is="renderDynamic(type, props)" />
</template>
```

---

## References

1. [https://nuxt.com](https://nuxt.com)
2. [https://vuejs.org](https://vuejs.org)
3. [https://nitro.unjs.io](https://nitro.unjs.io)
4. [https://vite.dev](https://vite.dev)
5. [https://unjs.io](https://unjs.io)
