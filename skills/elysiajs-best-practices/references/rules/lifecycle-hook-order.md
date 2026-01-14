---
title: Order Lifecycle Hooks Before Routes
impact: HIGH
impactDescription: Hooks only apply to subsequent routes
tags: lifecycle, hooks, routing, order
---

## Order Lifecycle Hooks Before Routes

**Impact: HIGH (Hooks only apply to subsequent routes)**

Lifecycle hooks only apply to routes registered after the hook declaration. Placing hooks after routes means those routes won't be affected by the hook logic.

**Incorrect (guard after routes):**

```typescript
app
  .get('/public', () => 'Public') // No auth check
  .post('/admin', () => 'Admin')  // No auth check
  .guard({
    beforeHandle: async ({ headers }) => {
      if (!headers.authorization) throw new Error('Unauthorized')
    }
  }, app => app
    .get('/protected', () => 'Protected') // Has auth check
  )
// Only /protected is guarded, /admin is exposed!
```

**Correct (guard before routes):**

```typescript
app
  .get('/public', () => 'Public') // No auth needed
  .guard({
    beforeHandle: async ({ headers }) => {
      if (!headers.authorization) throw new Error('Unauthorized')
    }
  }, app => app
    .post('/admin', () => 'Admin')        // Protected
    .get('/protected', () => 'Protected')  // Protected
  )
```

**Pattern: Layered hooks:**

```typescript
app
  // Global logging for all routes
  .onBeforeHandle({ as: 'global' }, ({ path }) => {
    console.log(`Request: ${path}`)
  })

  // Public routes
  .get('/health', () => 'OK')
  .get('/public', () => 'Public data')

  // Protected routes with auth
  .group('/api', app => app
    .onBeforeHandle(async ({ headers }) => {
      const user = await validateToken(headers.authorization)
      if (!user) throw new Error('Unauthorized')
    })
    .get('/users', () => getUsers())
    .post('/posts', ({ body }) => createPost(body))
  )

  // Admin routes with additional authorization
  .group('/admin', app => app
    .onBeforeHandle(async ({ headers }) => {
      const user = await validateToken(headers.authorization)
      if (!user?.isAdmin) throw new Error('Forbidden')
    })
    .delete('/users/:id', ({ params }) => deleteUser(params.id))
  )
```

**Execution order example:**

```typescript
new Elysia()
  .onRequest(() => console.log('1: Request received'))
  .get('/first', () => 'First') // Logs: 1

  .onTransform(() => console.log('2: Transform'))
  .get('/second', () => 'Second') // Logs: 1, 2

  .onBeforeHandle(() => console.log('3: Before handle'))
  .get('/third', () => 'Third') // Logs: 1, 2, 3
```

Reference: [ElysiaJS Lifecycle Documentation](https://elysiajs.com/essential/life-cycle)
