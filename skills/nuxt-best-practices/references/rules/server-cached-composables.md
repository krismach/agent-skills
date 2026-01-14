---
title: Use Cached Composables
impact: HIGH
impactDescription: Eliminates redundant database/API calls
tags: server, caching, performance
---

## Use Cached Composables

**Impact: HIGH**

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

**With Nitro storage:**

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

Reference: [Nitro Caching](https://nitro.unjs.io/guide/cache)
