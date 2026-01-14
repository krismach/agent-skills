---
title: Name Plugins for Automatic Deduplication
impact: CRITICAL
impactDescription: Prevents redundant plugin execution
tags: plugin, deduplication, performance, singleton
---

## Name Plugins for Automatic Deduplication

**Impact: CRITICAL (Prevents redundant plugin execution)**

ElysiaJS automatically deduplicates plugins with the same name, making them singletons. Without naming, the same plugin may execute multiple times, causing performance issues and unexpected behavior.

**Incorrect (unnamed plugin, executes multiple times):**

```typescript
const authPlugin = new Elysia()
  .derive(({ headers }) => ({
    user: validateToken(headers.authorization)
  }))

// Plugin executes multiple times
app.use(authPlugin)
app.group('/api', app => app.use(authPlugin))
// Auth logic runs twice per request in /api routes
```

**Correct (named plugin for deduplication):**

```typescript
const authPlugin = new Elysia({ name: 'auth' })
  .derive(({ headers }) => ({
    user: validateToken(headers.authorization)
  }))

app.use(authPlugin)
app.group('/api', app => app.use(authPlugin))
// Auth logic runs only once per request
```

**Advanced: Version-based deduplication with seed:**

```typescript
const authPlugin = (version: string) => new Elysia({
  name: 'auth',
  seed: version // Different seeds = different plugin instances
})
  .derive(({ headers }) => ({
    user: validateToken(headers.authorization)
  }))

// These are treated as different plugins
app.use(authPlugin('v1'))
app.use(authPlugin('v2'))
```

**Pattern for configurable plugins:**

```typescript
const rateLimitPlugin = (config: { limit: number }) =>
  new Elysia({
    name: 'rate-limit',
    seed: config // Different configs = different instances
  })
  .onBeforeHandle({ as: 'global' }, async ({ request }) => {
    await checkRateLimit(request, config.limit)
  })

app.use(rateLimitPlugin({ limit: 100 }))
```

Reference: [ElysiaJS Plugin Documentation](https://elysiajs.com/essential/plugin)
