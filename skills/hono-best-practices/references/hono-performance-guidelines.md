# Hono + Bun Performance Guidelines

Complete reference for building high-performance web applications with Hono framework on Bun runtime.

## Table of Contents

1. [Server Performance](#server-performance)
2. [Database & Caching](#database--caching)
3. [Routing & Middleware](#routing--middleware)
4. [Response Optimization](#response-optimization)
5. [Validation & Type Safety](#validation--type-safety)
6. [Static Files & Assets](#static-files--assets)
7. [Memory & Resource Management](#memory--resource-management)
8. [Development Patterns](#development-patterns)

## Architecture Overview

Hono + Bun provides:
- **Bun's JavaScriptCore engine** - Performance-focused JS engine from Safari
- **Native HTTP server** - Built-in, no need for separate libraries
- **Zero-dependency framework** - Hono uses only Web Standard APIs
- **Ultrafast routing** - RegExpRouter matches routes in single regex
- **Multi-runtime support** - Deploy anywhere (Cloudflare, Deno, Node.js, Bun)

## Server Performance

### Use Bun.serve() Directly

Bun's native server is optimized for performance. Use it instead of Node.js adapters.

```typescript
import { Hono } from 'hono'

const app = new Hono()

// Routes...

export default {
  port: 3000,
  fetch: app.fetch,
}
```

### Enable Connection Reuse

Configure server for optimal connection handling:

```typescript
export default {
  port: 3000,
  fetch: app.fetch,
  reusePort: true, // Enable SO_REUSEPORT
  development: false, // Disable in production for better performance
}
```

### Use Streaming for Large Responses

Stream responses to reduce memory usage and improve TTFB:

```typescript
app.get('/large-data', async (c) => {
  const stream = new ReadableStream({
    async start(controller) {
      for await (const chunk of dataSource) {
        controller.enqueue(chunk)
      }
      controller.close()
    }
  })
  return c.body(stream)
})
```

## Database & Caching

### Enable WAL Mode for SQLite

Write-Ahead Logging dramatically improves concurrent read/write performance:

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

// Enable WAL mode - do this once during setup
db.exec('PRAGMA journal_mode = WAL')
db.exec('PRAGMA synchronous = NORMAL') // Safe with WAL
db.exec('PRAGMA cache_size = -64000') // 64MB cache
db.exec('PRAGMA temp_store = MEMORY')
```

### Use Prepared Statements

Bun's SQLite driver caches prepared statements for massive performance gains:

```typescript
// Use .query() for automatic caching
const getUser = db.query('SELECT * FROM users WHERE id = ?')

// Execute multiple times efficiently
const user1 = getUser.get(1)
const user2 = getUser.get(2)

// Or use .prepare() for explicit control
const stmt = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)')
stmt.run('Alice', 'alice@example.com')
stmt.finalize() // Clean up when completely done
```

### Implement Query Result Caching

Cache frequently accessed data:

```typescript
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

// Usage
app.get('/stats', (c) => {
  const stats = getCached('stats', 60000, () => {
    return db.query('SELECT COUNT(*) as count FROM users').get()
  })
  return c.json(stats)
})
```

## Routing & Middleware

### Order Middleware by Execution Frequency

Place guards and authentication checks first:

```typescript
import { Hono } from 'hono'
import { logger } from 'hono/logger'
import { cors } from 'hono/cors'

const app = new Hono()

// 1. Security/auth checks (fail fast)
app.use('*', async (c, next) => {
  const token = c.req.header('Authorization')
  if (!token) return c.json({ error: 'Unauthorized' }, 401)
  await next()
})

// 2. CORS (needed for most requests)
app.use('*', cors())

// 3. Logging (least important)
app.use('*', logger())
```

### Use Specific Routes Before Wildcards

More specific routes should be defined first:

```typescript
// Correct order
app.get('/users/me', (c) => c.json({ user: 'current' }))
app.get('/users/:id', (c) => c.json({ id: c.req.param('id') }))

// Wrong order (would never match /users/me)
app.get('/users/:id', (c) => c.json({ id: c.req.param('id') }))
app.get('/users/me', (c) => c.json({ user: 'current' }))
```

### Combine Middleware

Use combine() to reduce middleware chain length:

```typescript
import { combine } from 'hono/combine'
import { logger } from 'hono/logger'
import { cors } from 'hono/cors'

// Combine multiple middleware into one
app.use('*', combine(
  logger(),
  cors(),
))
```

## Response Optimization

### Return Early on Validation Failure

Avoid unnecessary work when validation fails:

```typescript
// Bad - validates everything first
app.post('/users', async (c) => {
  const body = await c.req.json()
  const name = body.name
  const email = body.email
  const age = body.age

  if (!name || !email || !age) {
    return c.json({ error: 'Missing fields' }, 400)
  }

  // Process...
})

// Good - fail fast
app.post('/users', async (c) => {
  const body = await c.req.json()

  if (!body.name) return c.json({ error: 'Missing name' }, 400)
  if (!body.email) return c.json({ error: 'Missing email' }, 400)
  if (!body.age) return c.json({ error: 'Missing age' }, 400)

  // Process...
})
```

### Enable Compression

Use compression middleware for text responses:

```typescript
import { compress } from 'hono/compress'

app.use('*', compress())
```

### Set Cache Headers

Set appropriate cache headers for API responses:

```typescript
app.get('/data', (c) => {
  const data = fetchData()

  c.header('Cache-Control', 'public, max-age=300') // 5 minutes
  c.header('ETag', generateETag(data))

  return c.json(data)
})
```

## Validation & Type Safety

### Use Zod Validator

Zod provides runtime validation with excellent TypeScript integration:

```typescript
import { z } from 'zod'
import { zValidator } from '@hono/zod-validator'

const userSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(120),
})

app.post('/users', zValidator('json', userSchema), async (c) => {
  const user = c.req.valid('json') // Fully typed!
  // user.name, user.email, user.age are all validated
  return c.json({ success: true })
})
```

### Cache Validation Schemas

Reuse schema instances to avoid re-parsing:

```typescript
// Bad - creates new schema each request
app.post('/users', zValidator('json', z.object({
  name: z.string()
})), handler)

// Good - reuse schema
const userSchema = z.object({ name: z.string() })
app.post('/users', zValidator('json', userSchema), handler)
```

### Optimize RPC Type Inference

For large applications, compile RPC types to improve IDE performance:

```typescript
// In your client build
import type { AppType } from './server'
import { hc } from 'hono/client'

// This creates types at compile time
const client = hc<AppType>('http://localhost:3000')
```

## Static Files & Assets

### Use Bun.file() for Static Assets

Bun's native file API is highly optimized:

```typescript
import { serveStatic } from 'hono/bun'

app.use('/static/*', serveStatic({ root: './' }))

// Or custom implementation
app.get('/files/:filename', async (c) => {
  const filename = c.req.param('filename')
  const file = Bun.file(`./files/${filename}`)

  if (!await file.exists()) {
    return c.notFound()
  }

  return c.body(file.stream())
})
```

### Set Aggressive Caching for Immutable Assets

Assets with content hashes can be cached indefinitely:

```typescript
app.get('/static/:hash/:filename', async (c) => {
  const { filename } = c.req.param()
  const file = Bun.file(`./static/${filename}`)

  c.header('Cache-Control', 'public, max-age=31536000, immutable')
  return c.body(file.stream())
})
```

## Memory & Resource Management

### Finalize SQLite Statements

In performance-critical applications, explicitly finalize statements:

```typescript
const stmt = db.prepare('INSERT INTO logs (message) VALUES (?)')

for (const message of messages) {
  stmt.run(message)
}

stmt.finalize() // Free resources
```

### Use Streams for Large Files

Avoid loading entire files into memory:

```typescript
app.get('/download/:file', async (c) => {
  const filename = c.req.param('file')
  const file = Bun.file(`./downloads/${filename}`)

  // Stream instead of loading into memory
  c.header('Content-Type', file.type)
  c.header('Content-Length', file.size.toString())

  return c.body(file.stream())
})
```

### Implement Connection Pooling

For external databases, use connection pooling:

```typescript
import { Pool } from 'pg'

const pool = new Pool({
  max: 20, // Maximum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
})

app.get('/users', async (c) => {
  const client = await pool.connect()
  try {
    const result = await client.query('SELECT * FROM users')
    return c.json(result.rows)
  } finally {
    client.release() // Always release
  }
})
```

## Development Patterns

### Use Context Efficiently

Extract from context once, pass down:

```typescript
// Bad - repeated context access
app.post('/users', async (c) => {
  await validateUser(c)
  await saveUser(c)
  await sendEmail(c)
})

// Good - extract once
app.post('/users', async (c) => {
  const body = await c.req.json()
  const user = await validateUser(body)
  await saveUser(user)
  await sendEmail(user.email)
})
```

### Use JSX for Templating

Hono's JSX is faster than string concatenation:

```typescript
import { html } from 'hono/html'

// Using JSX
const Layout = (props: { children: any }) => {
  return (
    <html>
      <head>
        <title>My App</title>
      </head>
      <body>{props.children}</body>
    </html>
  )
}

app.get('/', (c) => {
  return c.html(
    <Layout>
      <h1>Hello World</h1>
    </Layout>
  )
})
```

### Implement Structured Logging

Use structured logs for better observability:

```typescript
import { logger } from 'hono/logger'

// Custom logger with structured output
const structuredLogger = async (c: Context, next: Next) => {
  const start = Date.now()
  await next()
  const duration = Date.now() - start

  console.log(JSON.stringify({
    method: c.req.method,
    path: c.req.path,
    status: c.res.status,
    duration,
    timestamp: new Date().toISOString(),
  }))
}

app.use('*', structuredLogger)
```

## Production Checklist

- [ ] WAL mode enabled for SQLite
- [ ] Prepared statements used for all queries
- [ ] Compression enabled for text responses
- [ ] Cache headers set appropriately
- [ ] Middleware ordered by execution frequency
- [ ] Static assets served with content hashing
- [ ] Connection pooling for external databases
- [ ] Structured logging implemented
- [ ] Error handling with proper cleanup
- [ ] Memory usage monitored
- [ ] TypeScript strict mode enabled
- [ ] Production environment variables set
