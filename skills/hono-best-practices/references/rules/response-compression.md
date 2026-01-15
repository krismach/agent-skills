---
title: Enable Response Compression
impact: HIGH
impactDescription: reduces bandwidth by 70-90% for text responses
tags: compression, response, bandwidth, performance
---

## Enable Response Compression

Use compression middleware to reduce response size for text-based content (JSON, HTML, CSS, JavaScript).

**Incorrect (no compression):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/data', (c) => {
  const largeData = generateLargeJson() // 500KB JSON
  return c.json(largeData) // Sends 500KB uncompressed
})
```

**Correct (with compression):**

```typescript
import { Hono } from 'hono'
import { compress } from 'hono/compress'

const app = new Hono()

// Enable compression globally
app.use('*', compress())

app.get('/data', (c) => {
  const largeData = generateLargeJson() // 500KB JSON
  return c.json(largeData) // Sends ~50KB compressed (90% reduction)
})
```

**Configure compression options:**

```typescript
import { compress } from 'hono/compress'

app.use('*', compress({
  encoding: 'gzip', // or 'deflate', 'br' (brotli)
}))
```

**Selective compression:**

```typescript
import { compress } from 'hono/compress'

// Only compress API routes
app.use('/api/*', compress())

// Don't compress already-compressed content
app.get('/images/*', (c) => {
  // JPEG/PNG are already compressed
  return serveImage(c)
})
```

**Custom compression logic:**

```typescript
import { compress } from 'hono/compress'

// Compress only if response is large enough
app.use('*', async (c, next) => {
  await next()

  const contentLength = c.res.headers.get('Content-Length')
  const shouldCompress = contentLength && parseInt(contentLength) > 1024

  if (shouldCompress) {
    // Apply compression
    return compress()(c, async () => {})
  }
})
```

**Avoid compressing binary content:**

```typescript
const shouldCompress = (contentType: string) => {
  const compressible = [
    'text/',
    'application/json',
    'application/javascript',
    'application/xml',
  ]
  return compressible.some(type => contentType.startsWith(type))
}

app.use('*', async (c, next) => {
  await next()

  const contentType = c.res.headers.get('Content-Type') || ''
  if (shouldCompress(contentType)) {
    return compress()(c, async () => {})
  }
})
```

Best practices:
- Enable compression globally for API routes
- Don't compress already-compressed formats (images, video, PDFs)
- Set minimum size threshold (e.g., 1KB)
- Use Brotli for static assets (best compression)
- Use Gzip for dynamic content (faster compression)
- Pre-compress static assets at build time

Reference: [Hono Compress Middleware](https://hono.dev/docs/middleware/builtin/compress)
