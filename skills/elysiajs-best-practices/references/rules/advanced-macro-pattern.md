---
title: Use Macros for Reusable Schema Logic
impact: LOW-MEDIUM
impactDescription: Type-safe configuration patterns
tags: advanced, macro, schema, reusability
---

## Use Macros for Reusable Schema Logic

**Impact: LOW-MEDIUM (Type-safe configuration patterns)**

Macros allow composing complex validation and transformation logic into simple, reusable configurations with full type safety. Use them for common patterns like pagination, authentication, or rate limiting.

**Incorrect (repeated configuration):**

```typescript
app
  .get('/users', ({ query }) => {
    const page = Math.max(1, query.page ?? 1)
    const limit = Math.min(100, Math.max(1, query.limit ?? 10))
    return getUsers(page, limit)
  }, {
    query: t.Object({
      page: t.Optional(t.Numeric()),
      limit: t.Optional(t.Numeric())
    })
  })

  .get('/posts', ({ query }) => {
    const page = Math.max(1, query.page ?? 1)
    const limit = Math.min(100, Math.max(1, query.limit ?? 10))
    return getPosts(page, limit)
  }, {
    query: t.Object({
      page: t.Optional(t.Numeric()),
      limit: t.Optional(t.Numeric())
    })
  })
// Repeated pagination logic
```

**Correct (using macro):**

```typescript
const plugin = new Elysia()
  .macro(({ onBeforeHandle }) => ({
    paginate(config: { maxLimit?: number; defaultLimit?: number }) {
      const maxLimit = config.maxLimit ?? 100
      const defaultLimit = config.defaultLimit ?? 10

      onBeforeHandle(({ query }) => {
        query.page = Math.max(1, query.page ?? 1)
        query.limit = Math.min(
          maxLimit,
          Math.max(1, query.limit ?? defaultLimit)
        )
      })
    }
  }))

app
  .use(plugin)

  .get('/users', ({ query }) => {
    // query.page and query.limit are sanitized
    return getUsers(query.page, query.limit)
  }, {
    paginate: { maxLimit: 50, defaultLimit: 10 },
    query: t.Object({
      page: t.Optional(t.Numeric()),
      limit: t.Optional(t.Numeric())
    })
  })

  .get('/posts', ({ query }) => {
    return getPosts(query.page, query.limit)
  }, {
    paginate: { maxLimit: 100, defaultLimit: 20 },
    query: t.Object({
      page: t.Optional(t.Numeric()),
      limit: t.Optional(t.Numeric())
    })
  })
```

**Pattern: Authentication macro:**

```typescript
const authPlugin = new Elysia()
  .macro(({ onBeforeHandle }) => ({
    auth(enabled: boolean) {
      if (!enabled) return

      onBeforeHandle(async ({ headers, set }) => {
        const token = headers.authorization?.replace('Bearer ', '')

        if (!token) {
          set.status = 401
          throw new Error('Unauthorized')
        }

        const user = await validateToken(token)
        if (!user) {
          set.status = 401
          throw new Error('Invalid token')
        }

        return { user }
      })
    }
  }))

app
  .use(authPlugin)

  .get('/public', () => 'Public') // No auth

  .get('/protected', ({ user }) => user, {
    auth: true // Automatically protected
  })

  .get('/admin', ({ user }) => {
    if (!user.isAdmin) throw new Error('Forbidden')
    return getAdminData()
  }, {
    auth: true
  })
```

**Advanced: Rate limiting macro:**

```typescript
import { LRUCache } from 'lru-cache'

const rateLimitCache = new LRUCache<string, number[]>({
  max: 10000,
  ttl: 1000 * 60 * 60 // 1 hour
})

const rateLimitPlugin = new Elysia()
  .macro(({ onBeforeHandle }) => ({
    rateLimit(config: {
      max: number
      window: number // milliseconds
      key?: 'ip' | 'user'
    }) {
      onBeforeHandle(({ request, set, user }) => {
        const key = config.key === 'user'
          ? user?.id ?? 'anonymous'
          : request.headers.get('x-forwarded-for') ?? 'unknown'

        const now = Date.now()
        const timestamps = rateLimitCache.get(key) ?? []

        // Filter timestamps within window
        const validTimestamps = timestamps.filter(
          t => now - t < config.window
        )

        if (validTimestamps.length >= config.max) {
          set.status = 429
          throw new Error('Rate limit exceeded')
        }

        validTimestamps.push(now)
        rateLimitCache.set(key, validTimestamps)
      })
    }
  }))

app
  .use(rateLimitPlugin)

  .get('/search', ({ query }) => search(query.q), {
    rateLimit: {
      max: 10,
      window: 60 * 1000, // 10 requests per minute
      key: 'ip'
    }
  })

  .post('/upload', ({ body, user }) => upload(body), {
    auth: true,
    rateLimit: {
      max: 5,
      window: 60 * 60 * 1000, // 5 uploads per hour
      key: 'user'
    }
  })
```

**Macro with schema transformation:**

```typescript
const plugin = new Elysia()
  .macro(({ onTransform }) => ({
    trimStrings(enabled: boolean) {
      if (!enabled) return

      onTransform(({ body }) => {
        if (typeof body === 'object' && body !== null) {
          for (const [key, value] of Object.entries(body)) {
            if (typeof value === 'string') {
              body[key] = value.trim()
            }
          }
        }
      })
    }
  }))

app
  .use(plugin)

  .post('/user', ({ body }) => createUser(body), {
    trimStrings: true,
    body: t.Object({
      name: t.String(),
      email: t.String()
    })
  })
```

**Composing multiple macros:**

```typescript
app.post('/api/posts', ({ body, user }) => {
  return createPost(user.id, body)
}, {
  auth: true,
  rateLimit: { max: 10, window: 60000 },
  paginate: { defaultLimit: 20 },
  trimStrings: true,
  body: t.Object({
    title: t.String(),
    content: t.String()
  })
})
```

Reference: [ElysiaJS Macro Documentation](https://elysiajs.com/patterns/macro)
