---
title: Use LRU Cache for Cross-Request Caching
impact: MEDIUM-HIGH
impactDescription: Reduces redundant computation
tags: performance, caching, memory, optimization
---

## Use LRU Cache for Cross-Request Caching

**Impact: MEDIUM-HIGH (Reduces redundant computation)**

Use an LRU (Least Recently Used) cache for expensive operations that can be shared across requests. This reduces database queries, API calls, and computation.

**Incorrect (no caching, repeated work):**

```typescript
app.get('/user/:id', async ({ params }) => {
  // Same user fetched on every request
  const user = await db.users.findUnique({ where: { id: params.id } })
  const profile = await buildUserProfile(user)
  return profile
})
```

**Correct (with LRU cache):**

```typescript
import { LRUCache } from 'lru-cache'

const userCache = new LRUCache<number, any>({
  max: 500, // Maximum entries
  ttl: 1000 * 60 * 5, // 5 minutes
  updateAgeOnGet: true
})

app.get('/user/:id', async ({ params }) => {
  const cached = userCache.get(params.id)
  if (cached) return cached

  const user = await db.users.findUnique({ where: { id: params.id } })
  const profile = await buildUserProfile(user)

  userCache.set(params.id, profile)
  return profile
}, {
  params: t.Object({
    id: t.Numeric()
  })
})
```

**Pattern: Cache plugin with invalidation:**

```typescript
import { LRUCache } from 'lru-cache'

const cachePlugin = <K, V>(name: string, options: {
  max: number
  ttl: number
}) => {
  const cache = new LRUCache<K, V>(options)

  return new Elysia({ name: `cache-${name}` })
    .decorate(name, {
      get: (key: K) => cache.get(key),
      set: (key: K, value: V) => cache.set(key, value),
      del: (key: K) => cache.delete(key),
      clear: () => cache.clear(),
      has: (key: K) => cache.has(key)
    })
}

app
  .use(cachePlugin<number, User>('userCache', {
    max: 1000,
    ttl: 1000 * 60 * 10 // 10 minutes
  }))

  .get('/user/:id', async ({ params, userCache }) => {
    const cached = userCache.get(params.id)
    if (cached) return cached

    const user = await fetchUser(params.id)
    userCache.set(params.id, user)
    return user
  })

  .put('/user/:id', async ({ params, body, userCache }) => {
    const user = await updateUser(params.id, body)
    // Invalidate cache on update
    userCache.del(params.id)
    return user
  })

  .delete('/user/:id', async ({ params, userCache }) => {
    await deleteUser(params.id)
    userCache.del(params.id)
    return { success: true }
  })
```

**Pattern: Memoization with cache:**

```typescript
import { LRUCache } from 'lru-cache'

const memoize = <Args extends any[], Result>(
  fn: (...args: Args) => Promise<Result>,
  options: { max: number; ttl: number }
) => {
  const cache = new LRUCache<string, Result>(options)

  return async (...args: Args): Promise<Result> => {
    const key = JSON.stringify(args)
    const cached = cache.get(key)
    if (cached) return cached

    const result = await fn(...args)
    cache.set(key, result)
    return result
  }
}

const getExpensiveData = memoize(
  async (userId: number, dateRange: string) => {
    // Expensive computation or API call
    return await computeAnalytics(userId, dateRange)
  },
  { max: 100, ttl: 1000 * 60 * 30 } // 30 minutes
)

app.get('/analytics/:userId', ({ params, query }) => {
  return getExpensiveData(params.userId, query.dateRange)
}, {
  params: t.Object({
    userId: t.Numeric()
  }),
  query: t.Object({
    dateRange: t.String()
  })
})
```

**Cache with stale-while-revalidate:**

```typescript
import { LRUCache } from 'lru-cache'

const cache = new LRUCache<string, any>({
  max: 500,
  ttl: 1000 * 60 * 5, // 5 min fresh
  allowStale: true,
  updateAgeOnGet: true
})

app.get('/stats', async () => {
  const cached = cache.get('stats')

  if (cached && !cache.getRemainingTTL('stats')) {
    // Stale data - return it but refresh in background
    refreshStats().then(fresh => cache.set('stats', fresh))
    return cached
  }

  if (cached) {
    // Fresh data
    return cached
  }

  // No cache - fetch and wait
  const stats = await refreshStats()
  cache.set('stats', stats)
  return stats
})

async function refreshStats() {
  return await db.stats.aggregate()
}
```

**Multi-level caching:**

```typescript
import { LRUCache } from 'lru-cache'

// Fast memory cache
const memCache = new LRUCache<string, any>({
  max: 100,
  ttl: 1000 * 60 // 1 minute
})

// Longer Redis cache
import { Redis } from 'ioredis'
const redis = new Redis()

app.get('/product/:id', async ({ params }) => {
  const key = `product:${params.id}`

  // Check L1 (memory)
  let product = memCache.get(key)
  if (product) return product

  // Check L2 (Redis)
  const cached = await redis.get(key)
  if (cached) {
    product = JSON.parse(cached)
    memCache.set(key, product)
    return product
  }

  // Fetch from DB
  product = await db.products.findUnique({ where: { id: params.id } })

  // Store in both caches
  memCache.set(key, product)
  await redis.setex(key, 300, JSON.stringify(product))

  return product
})
```

Reference: [LRU Cache Package](https://www.npmjs.com/package/lru-cache)
