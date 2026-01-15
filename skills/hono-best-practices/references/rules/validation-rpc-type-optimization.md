---
title: Optimize RPC Type Inference
impact: MEDIUM
impactDescription: improves IDE performance for large applications
tags: rpc, typescript, types, performance, ide
---

## Optimize RPC Type Inference

For large Hono applications using RPC, compile types ahead of time to improve IDE performance and reduce type instantiation overhead.

**Problem (slow IDE with large apps):**

```typescript
// server.ts
import { Hono } from 'hono'

const app = new Hono()

// 100+ routes...
app.get('/users', (c) => c.json({ users: [] }))
app.get('/posts', (c) => c.json({ posts: [] }))
// ... many more routes

export type AppType = typeof app

// client.ts
import { hc } from 'hono/client'
import type { AppType } from './server'

// Slow: IDE must infer types for all 100+ routes every time
const client = hc<AppType>('http://localhost:3000')
```

**Solution 1: Split apps by domain:**

```typescript
// users-app.ts
import { Hono } from 'hono'

export const usersApp = new Hono()
  .get('/', (c) => c.json({ users: [] }))
  .get('/:id', (c) => c.json({ user: {} }))
  .post('/', (c) => c.json({ created: true }))

// posts-app.ts
export const postsApp = new Hono()
  .get('/', (c) => c.json({ posts: [] }))
  .get('/:id', (c) => c.json({ post: {} }))

// server.ts
import { Hono } from 'hono'
import { usersApp } from './users-app'
import { postsApp } from './posts-app'

const app = new Hono()
  .route('/users', usersApp)
  .route('/posts', postsApp)

export default app
export type AppType = typeof app

// client.ts - much faster type inference
import { hc } from 'hono/client'
import type { AppType } from './server'

const client = hc<AppType>('http://localhost:3000')
```

**Solution 2: Pre-compile types:**

```typescript
// types.ts (generated at build time)
import type { Hono } from 'hono'
import type { app } from './server'

// Extract type once during build
export type AppType = typeof app

// client.ts
import type { AppType } from './types'
import { hc } from 'hono/client'

// Uses pre-compiled types
const client = hc<AppType>('http://localhost:3000')
```

**Solution 3: Use module-level route groups:**

```typescript
// Instead of one giant app, split into modules

// api/users/routes.ts
export const usersRoutes = new Hono()
  .get('/', handler1)
  .get('/:id', handler2)
  .post('/', handler3)

// api/posts/routes.ts
export const postsRoutes = new Hono()
  .get('/', handler1)
  .get('/:id', handler2)

// server.ts
const app = new Hono()
  .route('/api/users', usersRoutes)
  .route('/api/posts', postsRoutes)

// Client can import specific route types
import type { usersRoutes } from './api/users/routes'
const usersClient = hc<typeof usersRoutes>('/api/users')
```

**Best practices for large apps:**

```typescript
// 1. Group related routes
const apiV1 = new Hono()
  .route('/users', usersRoutes)
  .route('/posts', postsRoutes)

const apiV2 = new Hono()
  .route('/users', usersRoutesV2)
  .route('/posts', postsRoutesV2)

// 2. Compose gradually
const app = new Hono()
  .route('/api/v1', apiV1)
  .route('/api/v2', apiV2)

// 3. Export specific types
export type ApiV1Type = typeof apiV1
export type ApiV2Type = typeof apiV2
export type AppType = typeof app
```

**Compile-time type generation script:**

```typescript
// scripts/generate-types.ts
import { writeFileSync } from 'fs'
import type { app } from '../src/server'

// Generate type file
const typeDefinition = `
// Auto-generated - do not edit
import type { Hono } from 'hono'

export type AppType = typeof app

// Re-export for convenience
export type { app } from '../server'
`

writeFileSync('./src/types.generated.ts', typeDefinition)
console.log('Types generated!')

// Add to package.json:
// "scripts": {
//   "generate-types": "bun run scripts/generate-types.ts",
//   "build": "bun run generate-types && bun build ..."
// }
```

**Measuring improvement:**

```typescript
// Before: 5-10s for autocomplete with 100+ routes
// After: < 1s with proper splitting

// Check type complexity
// tsconfig.json
{
  "compilerOptions": {
    "noEmit": true,
    "skipLibCheck": true // Can speed up type checking
  }
}
```

When to optimize:
- ✅ More than 50 routes in single app
- ✅ IDE autocomplete taking > 2 seconds
- ✅ Large team with different domains
- ❌ Small apps (< 20 routes) - overhead not worth it

Reference: [Hono RPC](https://hono.dev/docs/guides/rpc)
