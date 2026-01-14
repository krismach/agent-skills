---
title: Use Method Chaining for Type Inference
impact: CRITICAL
impactDescription: Maintains TypeScript type safety
tags: plugin, typescript, chaining, types
---

## Use Method Chaining for Type Inference

**Impact: CRITICAL (Maintains TypeScript type safety)**

Every Elysia method returns a new type reference. Breaking the method chain loses accumulated type information, making properties inaccessible to TypeScript.

**Incorrect (breaking the chain):**

```typescript
const app = new Elysia()

// Type information is lost after each assignment
app.decorate('db', database)
app.state('version', '1.0')
app.get('/', ({ db, store }) => {
  // TypeScript error: db and store properties not recognized
  return { db, version: store.version }
})
```

**Correct (method chaining):**

```typescript
const app = new Elysia()
  .decorate('db', database)
  .state('version', '1.0')
  .get('/', ({ db, store }) => {
    // Types are properly inferred
    return { version: store.version }
  })
```

**Pattern: Chain-friendly plugin creation:**

```typescript
// Return this to maintain chainability
class DatabasePlugin {
  apply(app: Elysia) {
    return app
      .decorate('db', this.connection)
      .onStart(() => this.connect())
      .onStop(() => this.disconnect())
  }
}

const app = new Elysia()
  .use(new DatabasePlugin().apply)
  .get('/users', ({ db }) => db.users.findMany()) // Types work
```

**Acceptable: Intermediate variables for reuse:**

```typescript
// OK if you need to reuse the instance
const baseApp = new Elysia()
  .decorate('db', database)
  .state('version', '1.0')

// Extend the chain from the base
const apiApp = baseApp
  .group('/api', app => app
    .get('/users', ({ db }) => db.users.findMany())
  )

// Types are preserved
```

**Anti-pattern to avoid:**

```typescript
// DON'T: Multiple disconnected calls
const app = new Elysia()
app.use(plugin1)  // Returns new type
app.use(plugin2)  // Operating on original type
app.use(plugin3)  // Type information is fragmented

// DO: Single chain
const app = new Elysia()
  .use(plugin1)
  .use(plugin2)
  .use(plugin3)
```

Reference: [ElysiaJS Chaining Documentation](https://elysiajs.com/key-concept#method-chaining)
