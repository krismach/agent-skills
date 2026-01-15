---
title: Implement Query Result Caching
impact: HIGH
impactDescription: eliminates repeated database queries
tags: caching, database, performance, memory
---

## Implement Query Result Caching

Cache frequently accessed database query results to avoid repeated database hits.

**Incorrect (repeated queries):**

```typescript
import { Hono } from 'hono'
import { Database } from 'bun:sqlite'

const app = new Hono()
const db = new Database('app.db')

app.get('/stats', (c) => {
  // Queries database on every request
  const stats = db.query('SELECT COUNT(*) as count FROM users').get()
  return c.json(stats)
})

// 1000 requests = 1000 database queries for same data
```

**Correct (with caching):**

```typescript
import { Hono } from 'hono'
import { Database } from 'bun:sqlite'

const app = new Hono()
const db = new Database('app.db')

// Simple in-memory cache
const cache = new Map<string, { data: any, expires: number }>()

function getCached<T>(key: string, ttl: number, fn: () => T): T {
  const now = Date.now()
  const cached = cache.get(key)

  if (cached && cached.expires > now) {
    return cached.data
  }

  const data = fn()
  cache.set(key, { data, expires: now + ttl })
  return data
}

app.get('/stats', (c) => {
  const stats = getCached('stats', 60000, () => {
    return db.query('SELECT COUNT(*) as count FROM users').get()
  })
  return c.json(stats)
})

// 1000 requests in 1 minute = 1 database query
```

**Time-based cache with LRU eviction:**

```typescript
class LRUCache<T> {
  private cache = new Map<string, { data: T, expires: number }>()
  private maxSize: number

  constructor(maxSize = 100) {
    this.maxSize = maxSize
  }

  get(key: string): T | undefined {
    const item = this.cache.get(key)
    if (!item) return undefined

    if (item.expires < Date.now()) {
      this.cache.delete(key)
      return undefined
    }

    // Move to end (most recently used)
    this.cache.delete(key)
    this.cache.set(key, item)
    return item.data
  }

  set(key: string, data: T, ttl: number) {
    // Remove oldest if at capacity
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value
      this.cache.delete(firstKey)
    }

    this.cache.set(key, {
      data,
      expires: Date.now() + ttl
    })
  }
}

const queryCache = new LRUCache<any>(100)

function cachedQuery<T>(key: string, ttl: number, fn: () => T): T {
  const cached = queryCache.get(key)
  if (cached !== undefined) return cached

  const data = fn()
  queryCache.set(key, data, ttl)
  return data
}
```

**Cache invalidation on writes:**

```typescript
const cache = new Map<string, any>()

// Helper to invalidate cache
function invalidateCache(pattern: string) {
  for (const key of cache.keys()) {
    if (key.startsWith(pattern)) {
      cache.delete(key)
    }
  }
}

app.get('/users', (c) => {
  const users = getCached('users:all', 60000, () => {
    return db.query('SELECT * FROM users').all()
  })
  return c.json(users)
})

app.post('/users', async (c) => {
  const body = await c.req.json()
  db.query('INSERT INTO users (name) VALUES (?)').run(body.name)

  // Invalidate all user-related caches
  invalidateCache('users:')

  return c.json({ success: true })
})
```

**Tag-based cache invalidation:**

```typescript
class TaggedCache {
  private cache = new Map<string, any>()
  private tags = new Map<string, Set<string>>()

  set(key: string, value: any, tags: string[]) {
    this.cache.set(key, value)

    for (const tag of tags) {
      if (!this.tags.has(tag)) {
        this.tags.set(tag, new Set())
      }
      this.tags.get(tag)!.add(key)
    }
  }

  get(key: string) {
    return this.cache.get(key)
  }

  invalidateTag(tag: string) {
    const keys = this.tags.get(tag)
    if (!keys) return

    for (const key of keys) {
      this.cache.delete(key)
    }
    this.tags.delete(tag)
  }
}

const cache = new TaggedCache()

app.get('/users', (c) => {
  let users = cache.get('users:all')

  if (!users) {
    users = db.query('SELECT * FROM users').all()
    cache.set('users:all', users, ['users'])
  }

  return c.json(users)
})

app.post('/users', async (c) => {
  const body = await c.req.json()
  db.query('INSERT INTO users (name) VALUES (?)').run(body.name)

  // Invalidate all caches tagged with 'users'
  cache.invalidateTag('users')

  return c.json({ success: true })
})
```

**Per-user cache:**

```typescript
function getUserCacheKey(userId: number, resource: string): string {
  return `user:${userId}:${resource}`
}

app.get('/profile', async (c) => {
  const userId = c.get('userId')

  const profile = getCached(getUserCacheKey(userId, 'profile'), 300000, () => {
    return db.query('SELECT * FROM users WHERE id = ?').get(userId)
  })

  return c.json(profile)
})
```

**Cache with stale-while-revalidate:**

```typescript
interface CacheEntry<T> {
  data: T
  fresh: number  // Fresh until this time
  stale: number  // Stale until this time
  revalidating?: Promise<T>
}

async function getCachedWithSWR<T>(
  key: string,
  freshTTL: number,
  staleTTL: number,
  fn: () => Promise<T> | T
): Promise<T> {
  const now = Date.now()
  const cached = cache.get(key) as CacheEntry<T> | undefined

  // Return fresh cache
  if (cached && cached.fresh > now) {
    return cached.data
  }

  // Return stale while revalidating
  if (cached && cached.stale > now) {
    if (!cached.revalidating) {
      cached.revalidating = Promise.resolve(fn()).then(data => {
        cache.set(key, {
          data,
          fresh: now + freshTTL,
          stale: now + staleTTL,
        })
        return data
      })
    }
    return cached.data // Serve stale
  }

  // No cache or too old - fetch fresh
  const data = await fn()
  cache.set(key, {
    data,
    fresh: now + freshTTL,
    stale: now + staleTTL,
  })
  return data
}
```

**Automatic cache cleanup:**

```typescript
// Periodic cleanup of expired entries
setInterval(() => {
  const now = Date.now()
  for (const [key, value] of cache.entries()) {
    if (value.expires < now) {
      cache.delete(key)
    }
  }
}, 60000) // Clean every minute
```

Cache strategy guidelines:
- Short TTL (1-5 min): User-specific data
- Medium TTL (5-30 min): Semi-static data
- Long TTL (1+ hour): Reference data, statistics
- Always invalidate on writes
- Use LRU eviction to limit memory
- Monitor cache hit rates

Reference: [Caching Best Practices](https://hono.dev/docs/guides/best-practices)
