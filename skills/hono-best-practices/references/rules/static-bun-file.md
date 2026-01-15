---
title: Use Bun.file() for Static Assets
impact: MEDIUM
impactDescription: native file API optimized for performance
tags: static, files, bun, assets, performance
---

## Use Bun.file() for Static Assets

Bun's native file API is highly optimized for serving static files. Use it directly or through Hono's serveStatic middleware.

**Incorrect (reading entire file into memory):**

```typescript
import { Hono } from 'hono'
import { readFileSync } from 'fs'

const app = new Hono()

app.get('/files/:filename', async (c) => {
  const filename = c.req.param('filename')

  // Loads entire file into memory
  const content = readFileSync(`./public/${filename}`)

  return c.body(content)
})
```

**Correct (using Bun.file() with streaming):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/files/:filename', async (c) => {
  const filename = c.req.param('filename')
  const file = Bun.file(`./public/${filename}`)

  // Check if file exists
  if (!await file.exists()) {
    return c.notFound()
  }

  // Stream file efficiently
  c.header('Content-Type', file.type)
  c.header('Content-Length', file.size.toString())

  return c.body(file.stream())
})
```

**Using serveStatic middleware:**

```typescript
import { Hono } from 'hono'
import { serveStatic } from 'hono/bun'

const app = new Hono()

// Serve entire directory
app.use('/static/*', serveStatic({ root: './' }))

// With options
app.use('/public/*', serveStatic({
  root: './',
  rewriteRequestPath: (path) => path.replace(/^\/public/, '/assets'),
}))
```

**Optimized static file serving with caching:**

```typescript
app.get('/static/:filename', async (c) => {
  const filename = c.req.param('filename')
  const file = Bun.file(`./public/${filename}`)

  if (!await file.exists()) {
    return c.notFound()
  }

  // Generate ETag from file modification time and size
  const etag = `"${file.lastModified}-${file.size}"`
  const ifNoneMatch = c.req.header('If-None-Match')

  // Return 304 if client has current version
  if (ifNoneMatch === etag) {
    return c.body(null, 304)
  }

  // Set caching headers
  c.header('Content-Type', file.type)
  c.header('Content-Length', file.size.toString())
  c.header('ETag', etag)
  c.header('Cache-Control', 'public, max-age=3600') // 1 hour

  return c.body(file.stream())
})
```

**Immutable assets with content hashing:**

```typescript
// For files with content hash in name (e.g., app.abc123.js)
app.get('/assets/:hash/:filename', async (c) => {
  const { filename } = c.req.param()
  const file = Bun.file(`./dist/${filename}`)

  if (!await file.exists()) {
    return c.notFound()
  }

  // Aggressive caching for immutable assets
  c.header('Content-Type', file.type)
  c.header('Cache-Control', 'public, max-age=31536000, immutable')

  return c.body(file.stream())
})
```

**Range request support for large files:**

```typescript
app.get('/videos/:filename', async (c) => {
  const filename = c.req.param('filename')
  const file = Bun.file(`./videos/${filename}`)

  if (!await file.exists()) {
    return c.notFound()
  }

  const range = c.req.header('Range')

  if (!range) {
    // Full file
    c.header('Content-Type', file.type)
    c.header('Content-Length', file.size.toString())
    c.header('Accept-Ranges', 'bytes')
    return c.body(file.stream())
  }

  // Parse range header
  const [start, end] = range
    .replace(/bytes=/, '')
    .split('-')
    .map(Number)

  const chunkStart = start || 0
  const chunkEnd = end || file.size - 1
  const contentLength = chunkEnd - chunkStart + 1

  // Partial content
  c.header('Content-Type', file.type)
  c.header('Content-Length', contentLength.toString())
  c.header('Content-Range', `bytes ${chunkStart}-${chunkEnd}/${file.size}`)
  c.header('Accept-Ranges', 'bytes')

  return c.body(file.slice(chunkStart, chunkEnd + 1).stream(), 206)
})
```

Benefits:
- Native performance (no Node.js compatibility layer)
- Automatic MIME type detection
- Streaming support (constant memory)
- Built-in file metadata (size, lastModified)
- Zero-copy operations when possible

Reference: [Bun.file()](https://bun.com/docs/runtime)
