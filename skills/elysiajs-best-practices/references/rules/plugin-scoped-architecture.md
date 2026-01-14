---
title: Use Scoped Plugins for Feature Isolation
impact: HIGH
impactDescription: Better encapsulation and type safety
tags: plugin, scope, architecture, encapsulation
---

## Use Scoped Plugins for Feature Isolation

**Impact: HIGH (Better encapsulation and type safety)**

Use scoped plugins (default behavior) for feature-specific logic and `as: 'global'` only for cross-cutting concerns. This prevents unintended side effects and maintains better type boundaries.

**Incorrect (everything as global):**

```typescript
// CORS applied everywhere, even where not needed
const corsPlugin = new Elysia()
  .onBeforeHandle({ as: 'global' }, async ({ set }) => {
    set.headers['Access-Control-Allow-Origin'] = '*'
  })

// Database connection leaks to all instances
const dbPlugin = new Elysia()
  .decorate({ as: 'global' }, 'db', database)

app
  .use(corsPlugin)
  .use(dbPlugin)
  // Everything has db and CORS, even health checks
```

**Correct (scoped by default, global only when needed):**

```typescript
// CORS as global for all routes
const corsPlugin = new Elysia({ name: 'cors' })
  .onBeforeHandle({ as: 'global' }, async ({ set }) => {
    set.headers['Access-Control-Allow-Origin'] = '*'
  })

// Database scoped to routes that need it
const dbPlugin = new Elysia({ name: 'db' })
  .derive({ as: 'scoped' }, () => ({
    db: database
  }))

app
  .use(corsPlugin) // Global CORS
  .get('/health', () => 'OK') // No db access
  .group('/api', app => app
    .use(dbPlugin) // db only available in /api routes
    .get('/users', ({ db }) => db.users.findMany())
  )
```

**Pattern: Feature-based plugins:**

```typescript
// Each feature is a self-contained plugin
const userPlugin = new Elysia({ name: 'users', prefix: '/users' })
  .use(dbPlugin) // Explicitly declare dependencies
  .get('/', ({ db }) => db.users.findMany())
  .post('/', ({ body, db }) => db.users.create(body), {
    body: t.Object({
      name: t.String(),
      email: t.String({ format: 'email' })
    })
  })

const postPlugin = new Elysia({ name: 'posts', prefix: '/posts' })
  .use(dbPlugin)
  .get('/', ({ db }) => db.posts.findMany())

// Compose features
app
  .use(corsPlugin)
  .use(userPlugin)
  .use(postPlugin)
```

**Scope levels:**

```typescript
const plugin = new Elysia()
  // Local: Only this instance and child instances
  .derive({ as: 'local' }, () => ({ local: 'value' }))

  // Scoped (default): This instance, children, and one parent level
  .derive({ as: 'scoped' }, () => ({ scoped: 'value' }))

  // Global: Available everywhere
  .derive({ as: 'global' }, () => ({ global: 'value' }))
```

Reference: [ElysiaJS Scope Documentation](https://elysiajs.com/essential/plugin#scope)
