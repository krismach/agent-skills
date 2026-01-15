---
title: Enable Connection Reuse
impact: HIGH
impactDescription: reduces latency for subsequent requests
tags: server, http, connection, performance
---

## Enable Connection Reuse

Configure Bun's server for optimal connection handling with keep-alive and port reuse.

**Incorrect (basic configuration):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

export default {
  fetch: app.fetch,
  // Missing performance optimizations
}
```

**Correct (optimized configuration):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

export default {
  port: 3000,
  fetch: app.fetch,
  reusePort: true, // Enable SO_REUSEPORT for load balancing
  development: false, // Disable dev mode in production
}
```

**Full production configuration:**

```typescript
import { Hono } from 'hono'

const app = new Hono()

export default {
  port: parseInt(process.env.PORT || '3000'),
  hostname: '0.0.0.0', // Listen on all interfaces
  fetch: app.fetch,

  // Performance optimizations
  reusePort: true, // Allow multiple processes to bind to same port

  // Production settings
  development: false, // Disable dev mode logging and features

  // Error handling
  error(error) {
    console.error('Server error:', error)
    return new Response('Internal Server Error', { status: 500 })
  },
}
```

**Keep-Alive headers:**

```typescript
import { Hono } from 'hono'

const app = new Hono()

// Enable keep-alive for HTTP/1.1
app.use('*', async (c, next) => {
  await next()
  c.header('Connection', 'keep-alive')
  c.header('Keep-Alive', 'timeout=5, max=100') // 5s timeout, 100 requests
})
```

**Load balancing with reusePort:**

```typescript
// server.ts
import { Hono } from 'hono'

const app = new Hono()

export default {
  port: 3000,
  fetch: app.fetch,
  reusePort: true, // Multiple processes can bind to port 3000
}

// Start multiple processes for load balancing
// bun server.ts &
// bun server.ts &
// bun server.ts &
// Requests automatically balanced across processes
```

**Graceful shutdown pattern:**

```typescript
import { Hono } from 'hono'

const app = new Hono()

const server = Bun.serve({
  port: 3000,
  fetch: app.fetch,
  reusePort: true,
})

// Handle shutdown signals
process.on('SIGTERM', () => {
  console.log('SIGTERM received, closing server...')
  server.stop()
  // Close database connections
  db.close()
  process.exit(0)
})

process.on('SIGINT', () => {
  console.log('SIGINT received, closing server...')
  server.stop()
  db.close()
  process.exit(0)
})
```

**HTTP/2 support (when available):**

```typescript
// Bun automatically handles HTTP/2 when using TLS
export default {
  port: 443,
  fetch: app.fetch,

  // TLS enables HTTP/2
  tls: {
    key: Bun.file('./key.pem'),
    cert: Bun.file('./cert.pem'),
  },

  // HTTP/2 settings
  development: false,
  reusePort: true,
}
```

Benefits:
- **Keep-Alive**: Reuse TCP connections (reduces latency)
- **reusePort**: Enable load balancing across multiple processes
- **development: false**: Disable dev-mode overhead in production
- **Proper shutdown**: Clean up resources gracefully

Reference: [Bun.serve()](https://bun.com/docs/runtime)
