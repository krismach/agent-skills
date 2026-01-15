---
title: Use Bun.serve() Native Server
impact: CRITICAL
impactDescription: 2-4x better performance than Node.js adapters
tags: server, bun, performance, http
---

## Use Bun.serve() Native Server

Always use Bun's native server instead of Node.js adapters for optimal performance. Bun's HTTP server is built on native code with zero overhead.

**Incorrect (using Node.js adapter):**

```typescript
import { serve } from '@hono/node-server'
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hello!'))

serve({
  fetch: app.fetch,
  port: 3000,
})
```

**Correct (using Bun's native server):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hello!'))

export default {
  port: 3000,
  fetch: app.fetch,
}
```

**Advanced configuration:**

```typescript
export default {
  port: 3000,
  fetch: app.fetch,
  reusePort: true, // Enable SO_REUSEPORT for load balancing
  development: false, // Disable dev mode in production
}
```

Reference: [Bun HTTP Server](https://bun.com/docs/runtime)
