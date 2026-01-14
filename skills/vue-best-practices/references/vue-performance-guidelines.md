# Vue 3 & Nuxt 3 Performance Guidelines

Comprehensive performance optimization guide for Vue 3 and Nuxt 3 applications. 40+ rules organized by impact level.

## Table of Contents

1. [Eliminating Waterfalls (CRITICAL)](#1-eliminating-waterfalls-critical)
2. [Bundle Size Optimization (CRITICAL)](#2-bundle-size-optimization-critical)
3. [Server-Side Performance (HIGH)](#3-server-side-performance-high)
4. [Client-Side Data Fetching (MEDIUM-HIGH)](#4-client-side-data-fetching-medium-high)
5. [Reactivity Optimization (MEDIUM)](#5-reactivity-optimization-medium)
6. [Rendering Performance (MEDIUM)](#6-rendering-performance-medium)
7. [JavaScript Performance (LOW-MEDIUM)](#7-javascript-performance-low-medium)
8. [Advanced Patterns (LOW)](#8-advanced-patterns-low)

---

## 1. Eliminating Waterfalls (CRITICAL)

Waterfalls are the #1 performance killer. Each sequential await adds full network latency.

### 1.1 Defer Await Until Needed

Move `await` operations into branches where they're actually used.

**Incorrect:**
```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  const userData = await fetchUserData(userId) // Blocks both branches
  if (skipProcessing) return { skipped: true }
  return processUserData(userData)
}
```

**Correct:**
```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  if (skipProcessing) return { skipped: true }
  const userData = await fetchUserData(userId) // Only blocks when needed
  return processUserData(userData)
}
```

### 1.2 Promise.all() for Independent Operations

Execute independent async operations concurrently.

**Incorrect:**
```typescript
const user = await fetchUser()
const posts = await fetchPosts()
const comments = await fetchComments()
```

**Correct:**
```typescript
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments()
])
```

### 1.3 Strategic Suspense Boundaries

Show wrapper UI faster while data loads.

**Incorrect:**
```vue
<script setup lang="ts">
const data = await fetchData() // Blocks entire page
</script>

<template>
  <div>
    <div>Sidebar</div>
    <div>Header</div>
    <div><DataDisplay :data="data" /></div>
  </div>
</template>
```

**Correct:**
```vue
<template>
  <div>
    <div>Sidebar</div>
    <div>Header</div>
    <Suspense>
      <DataDisplay />
      <template #fallback><Skeleton /></template>
    </Suspense>
  </div>
</template>
```

### 1.4 Dependency-Based Parallelization with better-all

For complex dependency chains, use `better-all` to automatically maximize parallelism.

**Incorrect (manual orchestration):**
```typescript
const [post, config] = await Promise.all([fetchPost(postId), fetchConfig()])
const profile = await fetchProfile(post.authorId) // Config blocked unnecessarily
```

**Correct (automatic parallelization):**
```typescript
import { all } from 'better-all'

const { post, config, profile } = await all({
  async post() { return fetchPost(postId) },
  async config() { return fetchConfig() }, // Runs parallel with post
  async profile() {
    return fetchProfile((await this.$.post).authorId) // Parallel with config
  }
})
```

### 1.5 Parallelize API Route Dependencies

In Nuxt API routes, start independent operations concurrently.

**Correct:**
```typescript
// server/api/dashboard.get.ts
export default defineEventHandler(async (event) => {
  const userId = getQuery(event).userId as string

  const [user, posts, analytics] = await Promise.all([
    fetchUser(userId),
    fetchPosts(userId),
    fetchAnalytics(userId)
  ])

  return { user, posts, analytics }
})
```

---

## 2. Bundle Size Optimization (CRITICAL)

Reducing bundle size improves Time to Interactive and Largest Contentful Paint.

### 2.1 Avoid Barrel File Imports

Import directly from source files instead of barrel files.

**Incorrect:**
```typescript
import { Check, X, Menu } from 'lucide-vue-next' // Loads 1,583 modules
```

**Correct:**
```typescript
import Check from 'lucide-vue-next/dist/esm/icons/check'
import X from 'lucide-vue-next/dist/esm/icons/x'
import Menu from 'lucide-vue-next/dist/esm/icons/menu'
```

### 2.2 Dynamic Imports for Heavy Components

Use `defineAsyncComponent()` for large components.

**Incorrect:**
```vue
<script setup lang="ts">
import { MonacoEditor } from './monaco-editor' // Bundles ~300KB
</script>
```

**Correct:**
```vue
<script setup lang="ts">
const MonacoEditor = defineAsyncComponent(() =>
  import('./monaco-editor').then(m => m.MonacoEditor)
)
</script>
```

**In Nuxt 3:**
```vue
<template>
  <LazyMonacoEditor :value="code" />
</template>
```

### 2.3 Preload Based on User Intent

Preload heavy bundles before they're needed.

```vue
<script setup lang="ts">
const preloadEditor = () => {
  if (import.meta.client) {
    void import('./monaco-editor')
  }
}
</script>

<template>
  <button @mouseenter="preloadEditor" @click="openEditor">
    Open Editor
  </button>
</template>
```

### 2.4 Defer Non-Critical Third-Party Scripts

Load analytics and monitoring after main app is interactive.

```vue
<script setup lang="ts">
onMounted(() => {
  if (import.meta.client) {
    void import('@segment/analytics-next').then(({ default: Analytics }) => {
      Analytics.load({ writeKey: 'key' })
    })
  }
})
</script>
```

### 2.5 Conditional Imports for Platform-Specific Code

Use conditional imports to avoid bundling unused code.

**In Nuxt 3, use .client and .server suffixes:**
```
composables/
  useAnalytics.client.ts  # Only bundled for client
  useDatabase.server.ts   # Only bundled for server
```

---

## 3. Server-Side Performance (HIGH)

Optimize server-side rendering and data fetching.

### 3.1 Per-Request Deduplication with cachedFunction()

```typescript
// server/utils/auth.ts
export const getCurrentUser = cachedFunction(async () => {
  const session = await useAuth()
  if (!session?.user?.id) return null
  return await db.user.findUnique({ where: { id: session.user.id } })
}, {
  maxAge: 60 * 5,
  name: 'getCurrentUser'
})
```

### 3.2 Cross-Request LRU Caching

```typescript
// server/utils/cache.ts
export async function getCachedUser(id: string) {
  const storage = useStorage('cache')
  const cached = await storage.getItem(`user:${id}`)

  if (cached) return cached

  const user = await db.user.findUnique({ where: { id } })
  await storage.setItem(`user:${id}`, user, { ttl: 60 * 5 })

  return user
}
```

### 3.3 Parallel Data Fetching with Composables

```vue
<script setup lang="ts">
// Start both fetches immediately
const headerPromise = useFetch('/api/header')
const itemsPromise = useFetch('/api/sidebar')

// Await together
const [{ data: header }, { data: items }] = await Promise.all([
  headerPromise,
  itemsPromise
])
</script>
```

### 3.4 Minimize Serialization at SSR Boundaries

Only pass fields the client needs.

**Incorrect:**
```vue
<script setup lang="ts">
const { data: user } = await useFetch('/api/user') // 50 fields
</script>

<template>
  <Profile :user="user" /> <!-- passes entire object -->
</template>
```

**Correct:**
```vue
<script setup lang="ts">
const { data: user } = await useFetch('/api/user', {
  transform: (data) => ({ name: data.name }) // Pick only needed fields
})
</script>

<template>
  <Profile :name="user.name" />
</template>
```

---

## 4. Client-Side Data Fetching (MEDIUM-HIGH)

Efficient data fetching patterns reduce redundant network requests.

### 4.1 Use useFetch for Automatic Deduplication

```vue
<script setup lang="ts">
const { data: users } = await useFetch('/api/users', {
  key: 'users-list' // Multiple instances share one request
})
</script>
```

### 4.2 Deduplicate Global Event Listeners

Create shared composables for event listeners.

```typescript
// composables/useKeyboardShortcut.ts
import { useMagicKeys } from '@vueuse/core'

export function useKeyboardShortcut(key: string, callback: () => void) {
  const keys = useMagicKeys()
  const cmdKey = keys[`Cmd+${key}`]

  watch(cmdKey, (pressed) => {
    if (pressed) callback()
  })
}
```

---

## 5. Reactivity Optimization (MEDIUM)

Optimize Vue's reactivity system for better performance.

### 5.1 Use Computed for Derived State

```vue
<script setup lang="ts">
const items = ref<Item[]>([])
const query = ref('')

// Caches result, only recomputes when dependencies change
const filteredItems = computed(() => {
  return items.value.filter(item =>
    item.name.toLowerCase().includes(query.value.toLowerCase())
  )
})
</script>
```

### 5.2 Use shallowRef for Large Data Structures

```vue
<script setup lang="ts">
// Only top-level reference is reactive
const data = shallowRef({
  items: Array(10000).fill(0).map((_, i) => ({ id: i, name: `Item ${i}` }))
})

// Replace entire reference to trigger update
const updateItem = (index: number) => {
  const newData = { ...data.value }
  newData.items[index].count++
  data.value = newData
}
</script>
```

### 5.3 Specify Watch Dependencies Explicitly

Use explicit watch sources instead of `watchEffect`.

```vue
<script setup lang="ts">
const user = ref<User>()

// Only runs when user changes, not other reactive values
watch(user, async (newUser) => {
  if (newUser?.id) {
    await fetchUserData(newUser.id)
  }
})
</script>
```

### 5.4 Defer Reactive Reads to Usage Point

Read reactive values only when needed.

```vue
<script setup lang="ts">
const loading = ref(true)
const user = ref<User>()

// Only accesses user when not loading
const displayName = computed(() => {
  if (loading.value) return 'Loading...'
  return user.value?.name || 'Unknown'
})
</script>
```

### 5.5 Use v-once for Static Content

```vue
<template>
  <header v-once>
    <h1>{{ STATIC_CONFIG.title }}</h1>
    <p>Version: {{ STATIC_CONFIG.version }}</p>
  </header>
</template>
```

### 5.6 Subscribe to Derived Boolean State

```vue
<script setup lang="ts">
import { useMediaQuery } from '@vueuse/core'

// Only updates when crossing breakpoint
const isMobile = useMediaQuery('(max-width: 767px)')
</script>
```

---

## 6. Rendering Performance (MEDIUM)

Optimize the rendering process to reduce browser work.

### 6.1 Use v-if/v-else for Explicit Conditional Rendering

```vue
<template>
  <span v-if="count > 0" class="badge">{{ count }}</span>
  <span v-else class="badge-empty">No items</span>
</template>
```

### 6.2 Use v-show for Frequently Toggled Elements

```vue
<template>
  <!-- Component stays mounted, only CSS changes -->
  <HeavyComponent v-show="isVisible" />
</template>
```

### 6.3 CSS content-visibility for Long Lists

```vue
<template>
  <div class="overflow-y-auto h-screen">
    <div v-for="msg in messages" :key="msg.id" class="message-item">
      <Avatar :user="msg.author" />
      <div>{{ msg.content }}</div>
    </div>
  </div>
</template>

<style scoped>
.message-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;
}
</style>
```

### 6.4 Prevent Hydration Mismatch Without Flickering

```vue
<script setup lang="ts">
const theme = ref('light')

if (process.client) {
  theme.value = localStorage.getItem('theme') || 'light'
}
</script>

<template>
  <ClientOnly>
    <div :class="theme"><slot /></div>
    <template #fallback>
      <div class="light"><slot /></div>
    </template>
  </ClientOnly>
</template>
```

### 6.5 Animate SVG Wrappers, Not SVG Elements

```vue
<template>
  <div class="icon-wrapper animate-spin">
    <svg class="icon">
      <path d="..." />
    </svg>
  </div>
</template>
```

### 6.6 Use Stable Keys for List Rendering

```vue
<script setup lang="ts">
const items = ref([
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
])
</script>

<template>
  <div v-for="item in items" :key="item.id">
    <ExpensiveComponent :name="item.name" />
  </div>
</template>
```

### 6.7 Limit SVG Path Precision

Reduce decimal precision in SVG paths to decrease file size.

```vue
<script setup lang="ts">
const pathData = computed(() => {
  return points.value
    .map(([x, y], i) => {
      const cmd = i === 0 ? 'M' : 'L'
      return `${cmd} ${x.toFixed(2)} ${y.toFixed(2)}`
    })
    .join(' ')
})
</script>
```

---

## 7. JavaScript Performance (LOW-MEDIUM)

Micro-optimizations for hot paths.

### 7.1 Batch DOM CSS Changes

```typescript
// Correct: batch via class
element.classList.add('theme-dark')
```

### 7.2 Build Index Maps for Repeated Lookups

```typescript
const userMap = new Map(users.map(u => [u.id, u]))
const user = userMap.get(userId) // O(1) lookup
```

### 7.3 Cache Repeated Function Calls

```vue
<script setup lang="ts">
const expensiveResult = computed(() => expensiveFunction(input.value))
</script>
```

### 7.4 Use Set/Map for Lookups

```typescript
const activeIds = new Set(items.map(i => i.id))
const isActive = activeIds.has(id) // O(1) vs O(n)
```

### 7.5 Early Length Check for Array Comparisons

```typescript
if (a.length !== b.length) return false
```

### 7.6 Use toSorted() for Immutability

```typescript
const sorted = items.toSorted((a, b) => a.value - b.value)
```

---

## 8. Advanced Patterns (LOW)

Advanced patterns for specific cases requiring careful implementation.

### 8.1 Store Event Handlers in Refs

```typescript
export function useWindowEvent(event: string, handler: () => void) {
  const handlerRef = ref(handler)

  watch(() => handler, (newHandler) => {
    handlerRef.value = newHandler
  })

  onMounted(() => {
    const listener = (...args: any[]) => handlerRef.value(...args)
    window.addEventListener(event, listener)

    onUnmounted(() => {
      window.removeEventListener(event, listener)
    })
  })
}
```

### 8.2 useLatestValue for Stable Callback Refs

```typescript
export function useLatestValue<T>(value: T | Ref<T>) {
  const valueRef = ref(unref(value)) as Ref<T>

  watch(() => unref(value), (newValue) => {
    valueRef.value = newValue
  })

  return valueRef
}
```

---

## Priority Implementation Order

1. **CRITICAL (Apply First)**
   - Eliminate waterfalls with Promise.all()
   - Avoid barrel imports
   - Use dynamic imports for heavy components

2. **HIGH (Second Priority)**
   - Use cachedFunction() for server deduplication
   - Parallelize server data fetching
   - Minimize SSR serialization

3. **MEDIUM (Regular Optimization)**
   - Use computed for derived state
   - Apply v-show for toggles
   - Use explicit watch dependencies

4. **LOW (Fine-Tuning)**
   - JavaScript micro-optimizations
   - Advanced patterns as needed

## References

- [Vue 3 Documentation](https://vuejs.org)
- [Nuxt 3 Documentation](https://nuxt.com)
- [VueUse Composables](https://vueuse.org)
- [Vite Build Optimizations](https://vitejs.dev)
