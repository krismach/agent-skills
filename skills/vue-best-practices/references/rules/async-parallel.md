---
title: Promise.all() for Independent Operations
impact: CRITICAL
impactDescription: 2-10× improvement
tags: async, parallelization, promises, waterfalls
---

## Promise.all() for Independent Operations

When async operations have no interdependencies, execute them concurrently using `Promise.all()`.

**Incorrect (sequential execution, 3 round trips):**

```typescript
const user = await fetchUser()
const posts = await fetchPosts()
const comments = await fetchComments()
```

**Correct (parallel execution, 1 round trip):**

```typescript
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments()
])
```

**In Vue composables:**

```typescript
// Incorrect
export async function useUserDashboard(userId: string) {
  const profile = await fetchProfile(userId)
  const activities = await fetchActivities(userId)
  const notifications = await fetchNotifications(userId)

  return { profile, activities, notifications }
}

// Correct
export async function useUserDashboard(userId: string) {
  const [profile, activities, notifications] = await Promise.all([
    fetchProfile(userId),
    fetchActivities(userId),
    fetchNotifications(userId)
  ])

  return { profile, activities, notifications }
}
```

Reference: [Vue 3 Async Components](https://vuejs.org/guide/components/async.html)
