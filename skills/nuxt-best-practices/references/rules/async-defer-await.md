---
title: Defer Await Until Needed
impact: CRITICAL
impactDescription: Eliminates unnecessary blocking in async functions
tags: async, waterfalls, server
---

## Defer Await Until Needed

**Impact: CRITICAL**

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

This optimization is especially valuable when the skipped branch is frequently taken, or when the deferred operation is expensive.

Reference: [Nuxt Data Fetching](https://nuxt.com/docs/getting-started/data-fetching)
