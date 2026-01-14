---
title: Cross-Request LRU Caching
impact: HIGH
impactDescription: caches across requests
tags: server, cache, lru, cross-request
---

## Cross-Request LRU Caching

`cachedFunction()` only works within one request. For data shared across sequential requests, use an LRU cache or Nuxt's built-in storage layer.

**Implementation with Nuxt Storage:**

```typescript
// server/utils/cache.ts
export async function getCachedUser(id: string) {
  const storage = useStorage('cache')
  const cached = await storage.getItem(`user:${id}`)

  if (cached) return cached

  const user = await db.user.findUnique({ where: { id } })
  await storage.setItem(`user:${id}`, user, {
    ttl: 60 * 5 // 5 minutes
  })

  return user
}
```

**Implementation with LRU cache:**

```typescript
// server/utils/lru-cache.ts
import { LRUCache } from 'lru-cache'

const cache = new LRUCache<string, any>({
  max: 1000,
  ttl: 5 * 60 * 1000  // 5 minutes
})

export async function getUser(id: string) {
  const cached = cache.get(id)
  if (cached) return cached

  const user = await db.user.findUnique({ where: { id } })
  cache.set(id, user)
  return user
}

// Request 1: DB query, result cached
// Request 2: cache hit, no DB query
```

**Using with Nitro storage:**

```typescript
// server/api/user/[id].get.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  const storage = useStorage('cache')

  const cacheKey = `user:${id}`
  const cached = await storage.getItem(cacheKey)

  if (cached) {
    return cached
  }

  const user = await db.user.findUnique({ where: { id } })
  await storage.setItem(cacheKey, user, { ttl: 300 })

  return user
})
```

Use when sequential user actions hit multiple endpoints needing the same data within seconds. In serverless, consider Redis for cross-process caching.

Reference: [Nitro Storage Layer](https://nitro.unjs.io/guide/storage)
