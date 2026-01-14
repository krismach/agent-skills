---
title: Parallel Data Fetching with Promise.all()
impact: CRITICAL
impactDescription: Reduces waterfall latency by 2-3x
tags: async, waterfalls, promise
---

## Parallel Data Fetching with Promise.all()

**Impact: CRITICAL**

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

Reference: [JavaScript Promise.all()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
