---
title: Define Specific Routes Before Wildcards
impact: HIGH
impactDescription: prevents route shadowing and incorrect matching
tags: routing, patterns, hono, order
---

## Define Specific Routes Before Wildcards

Hono's router matches routes in the order they're defined. Always define more specific routes before parameterized or wildcard routes.

**Incorrect (specific route is shadowed):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

// This matches ALL /users/* paths including /users/me
app.get('/users/:id', (c) => {
  const id = c.req.param('id') // Will be "me" for /users/me
  return c.json({ id })
})

// This will NEVER match - shadowed by the route above
app.get('/users/me', (c) => {
  return c.json({ user: 'current user' })
})
```

**Correct (specific routes first):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

// Specific routes first
app.get('/users/me', (c) => {
  return c.json({ user: 'current user' })
})

app.get('/users/stats', (c) => {
  return c.json({ stats: 'user statistics' })
})

// Parameterized routes after
app.get('/users/:id', (c) => {
  const id = c.req.param('id')
  return c.json({ id })
})
```

**Multiple levels of specificity:**

```typescript
// Most specific first
app.get('/api/v2/users/me/settings', handler)
app.get('/api/v2/users/me', handler)
app.get('/api/v2/users/:id/settings', handler)
app.get('/api/v2/users/:id', handler)
app.get('/api/v2/*', handler) // Catch-all last
```

**Wrong order causes bugs:**

```typescript
// BAD: Wildcard first
app.get('/api/*', wildcardHandler)
app.get('/api/health', healthHandler) // Never reached!

// GOOD: Specific first
app.get('/api/health', healthHandler)
app.get('/api/*', wildcardHandler)
```

Route ordering best practices:
1. Exact string matches first
2. Longer paths before shorter
3. Parameterized routes (`:id`) in the middle
4. Wildcard routes (`*`) last
5. Within same specificity, order by frequency

Reference: [Hono Routing](https://hono.dev/docs/api/routing)
