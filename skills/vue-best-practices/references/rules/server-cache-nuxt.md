---
title: Per-Request Deduplication with cachedFunction()
impact: HIGH
impactDescription: deduplicates within request
tags: server, cache, nuxt-cache, deduplication
---

## Per-Request Deduplication with cachedFunction()

Use `cachedFunction()` for server-side request deduplication in Nuxt 3. Authentication and database queries benefit most.

**Usage:**

```typescript
// server/utils/auth.ts
export const getCurrentUser = cachedFunction(async () => {
  const session = await useAuth()
  if (!session?.user?.id) return null
  return await db.user.findUnique({
    where: { id: session.user.id }
  })
}, {
  maxAge: 60 * 5, // 5 minutes
  name: 'getCurrentUser',
  getKey: () => 'current-user'
})
```

Within a single request, multiple calls to `getCurrentUser()` execute the query only once.

**In API routes:**

```typescript
// server/api/user/profile.get.ts
export default defineCachedEventHandler(async (event) => {
  const userId = getQuery(event).userId as string

  return await db.user.findUnique({
    where: { id: userId }
  })
}, {
  maxAge: 60 * 5, // Cache for 5 minutes
  getKey: (event) => {
    const userId = getQuery(event).userId as string
    return `user:${userId}`
  }
})
```

**With composables:**

```typescript
// composables/useUser.ts
export async function useCurrentUser() {
  const { data } = await useFetch('/api/user/current', {
    key: 'current-user',
    getCachedData: (key) => useNuxtApp().payload.data[key]
  })

  return data
}
```

Reference: [Nuxt 3 Data Fetching](https://nuxt.com/docs/getting-started/data-fetching)
