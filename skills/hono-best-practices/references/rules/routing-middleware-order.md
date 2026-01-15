---
title: Order Middleware by Execution Frequency
impact: HIGH
impactDescription: reduces unnecessary processing on failed requests
tags: middleware, routing, performance, optimization
---

## Order Middleware by Execution Frequency

Place authentication and validation middleware before logging and other low-priority middleware. This allows requests to fail fast without executing unnecessary code.

**Incorrect (executes all middleware even for unauthorized requests):**

```typescript
import { Hono } from 'hono'
import { logger } from 'hono/logger'
import { cors } from 'hono/cors'
import { compress } from 'hono/compress'

const app = new Hono()

// Expensive operations run first
app.use('*', logger())
app.use('*', compress())
app.use('*', cors())

// Auth check last - but most requests fail here
app.use('*', async (c, next) => {
  const token = c.req.header('Authorization')
  if (!token) return c.json({ error: 'Unauthorized' }, 401)
  // Validate token...
  await next()
})
```

**Correct (fail fast, then optimize successful requests):**

```typescript
import { Hono } from 'hono'
import { logger } from 'hono/logger'
import { cors } from 'hono/cors'
import { compress } from 'hono/compress'

const app = new Hono()

// 1. Authentication - fail fast (40% of requests fail here)
app.use('*', async (c, next) => {
  const token = c.req.header('Authorization')
  if (!token) return c.json({ error: 'Unauthorized' }, 401)
  // Validate token...
  await next()
})

// 2. Rate limiting - prevent abuse early
app.use('*', async (c, next) => {
  const ip = c.req.header('X-Forwarded-For')
  if (isRateLimited(ip)) return c.json({ error: 'Too many requests' }, 429)
  await next()
})

// 3. CORS - needed for most requests
app.use('*', cors())

// 4. Compression - optimize successful responses
app.use('*', compress())

// 5. Logging - least critical, runs last
app.use('*', logger())
```

**Using combine() for performance:**

```typescript
import { combine } from 'hono/combine'

// Combine low-impact middleware to reduce chain length
app.use('*', combine(
  cors(),
  compress(),
  logger(),
))
```

Priority order:
1. **Auth/Security** - Most requests should fail here
2. **Rate limiting** - Prevent abuse early
3. **Validation** - Check request format
4. **CORS** - Cross-origin headers
5. **Compression** - Optimize responses
6. **Logging** - Observability (least critical)

Reference: [Hono Middleware](https://hono.dev/docs/guides/middleware)
