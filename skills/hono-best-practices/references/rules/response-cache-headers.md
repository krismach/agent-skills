---
title: Set Proper Cache Headers
impact: HIGH
impactDescription: reduces server load and improves response times
tags: caching, headers, http, performance
---

## Set Proper Cache Headers

Set appropriate cache headers to reduce server load and improve response times for both static and dynamic content.

**Incorrect (no caching):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/api/data', (c) => {
  const data = fetchData()
  return c.json(data) // No cache headers - server hit every time
})

app.get('/static/logo.png', async (c) => {
  const file = Bun.file('./assets/logo.png')
  return c.body(file.stream()) // No caching
})
```

**Correct (appropriate caching):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

// Dynamic content - short cache
app.get('/api/data', (c) => {
  const data = fetchData()

  c.header('Cache-Control', 'public, max-age=300') // 5 minutes
  c.header('Vary', 'Accept-Encoding')

  return c.json(data)
})

// Static content - long cache with ETag
app.get('/static/:filename', async (c) => {
  const filename = c.req.param('filename')
  const file = Bun.file(`./assets/${filename}`)

  const etag = `"${file.lastModified}-${file.size}"`
  const ifNoneMatch = c.req.header('If-None-Match')

  // Return 304 if cached version is still valid
  if (ifNoneMatch === etag) {
    return c.body(null, 304)
  }

  c.header('Cache-Control', 'public, max-age=31536000') // 1 year
  c.header('ETag', etag)

  return c.body(file.stream())
})
```

**Immutable assets (with content hash):**

```typescript
// For assets with content hash in URL (e.g., app.abc123.js)
app.get('/assets/:hash/:filename', async (c) => {
  const { filename } = c.req.param()
  const file = Bun.file(`./dist/${filename}`)

  if (!await file.exists()) {
    return c.notFound()
  }

  // Aggressive caching for immutable content
  c.header('Cache-Control', 'public, max-age=31536000, immutable')
  c.header('Content-Type', file.type)

  return c.body(file.stream())
})
```

**Conditional caching based on authentication:**

```typescript
app.get('/api/profile', async (c) => {
  const user = c.get('user')

  // Private cache for authenticated users
  c.header('Cache-Control', 'private, max-age=300')
  c.header('Vary', 'Authorization')

  return c.json({ profile: user })
})

app.get('/api/public-stats', (c) => {
  const stats = getStats()

  // Public cache for anonymous content
  c.header('Cache-Control', 'public, max-age=600')

  return c.json(stats)
})
```

**Stale-while-revalidate pattern:**

```typescript
app.get('/api/news', (c) => {
  const news = fetchNews()

  // Serve stale content while revalidating in background
  c.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=300')

  return c.json(news)
})
```

**No caching for sensitive data:**

```typescript
app.post('/api/login', async (c) => {
  const result = await login(c)

  // Prevent caching of sensitive responses
  c.header('Cache-Control', 'no-store, no-cache, must-revalidate')
  c.header('Pragma', 'no-cache')
  c.header('Expires', '0')

  return c.json(result)
})
```

**Middleware for automatic cache headers:**

```typescript
const cacheControl = (directive: string) => {
  return async (c: Context, next: Next) => {
    await next()
    c.header('Cache-Control', directive)
  }
}

// Apply to routes
app.get('/api/static', cacheControl('public, max-age=3600'), handler)
app.get('/api/dynamic', cacheControl('public, max-age=60'), handler)
```

**Cache strategy reference:**

```typescript
// Immutable (with hash): Cache forever
c.header('Cache-Control', 'public, max-age=31536000, immutable')

// Static content: Long cache with revalidation
c.header('Cache-Control', 'public, max-age=86400')
c.header('ETag', etag)

// API responses: Short cache
c.header('Cache-Control', 'public, max-age=300')

// User-specific: Private cache
c.header('Cache-Control', 'private, max-age=300')

// Real-time: No cache
c.header('Cache-Control', 'no-cache')

// Sensitive: Never cache
c.header('Cache-Control', 'no-store')
```

**CDN optimization:**

```typescript
app.get('/api/popular-items', (c) => {
  const items = getPopularItems()

  // Cache at CDN and browser
  c.header('Cache-Control', 'public, max-age=300, s-maxage=3600')
  c.header('CDN-Cache-Control', 'max-age=3600')

  return c.json(items)
})
```

Cache directive meanings:
- `public` - Can be cached by any cache
- `private` - Only user's browser can cache
- `no-cache` - Must revalidate before use
- `no-store` - Never cache (sensitive data)
- `max-age=N` - Cache for N seconds
- `s-maxage=N` - CDN cache time
- `immutable` - Never changes (with hash)
- `stale-while-revalidate=N` - Serve stale for N seconds while updating

Reference: [MDN Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
