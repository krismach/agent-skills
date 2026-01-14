# ElysiaJS Performance Guidelines

Complete guide to writing high-performance ElysiaJS applications. Rules are organized by category and prioritized by impact.

## Table of Contents

1. [Schema & Validation](#1-schema--validation-critical)
2. [Plugin Architecture](#2-plugin-architecture-critical)
3. [Lifecycle Optimization](#3-lifecycle-optimization-high)
4. [Async Patterns](#4-async-patterns-high)
5. [Error Handling](#5-error-handling-medium-high)
6. [Response Optimization](#6-response-optimization-medium)
7. [Memory & Performance](#7-memory--performance-medium)
8. [Advanced Patterns](#8-advanced-patterns-low-medium)

---

## 1. Schema & Validation (CRITICAL)

### Use TypeBox for Schema Validation

ElysiaJS uses TypeBox with AOT compilation in Bun, providing **18× faster validation** than alternatives like Zod. A single TypeBox schema provides:
- Runtime validation
- Type coercion
- TypeScript types
- OpenAPI documentation

```typescript
// ✅ CORRECT: TypeBox with automatic type inference
app.post('/user', ({ body }) => createUser(body), {
  body: t.Object({
    name: t.String(),
    email: t.String({ format: 'email' }),
    age: t.Number({ minimum: 18 })
  })
})

// ❌ INCORRECT: External validation
import { z } from 'zod'
app.post('/user', async ({ body }) => {
  const schema = z.object({ name: z.string() })
  const validated = schema.parse(body) // Slower, no auto types
})
```

### Use t.Numeric() for Numeric Strings

Query and path parameters are always strings. Use `t.Numeric()` for automatic coercion:

```typescript
app.get('/user/:id', ({ params }) => getUserById(params.id), {
  params: t.Object({
    id: t.Numeric({ minimum: 1 })
  })
})
```

### Define Schemas Inline for Type Inference

Elysia requires inline schemas or const assertions for proper type inference:

```typescript
// ✅ CORRECT: Inline definition
app.post('/user', ({ body }) => body, {
  body: t.Object({ name: t.String() })
})

// ❌ INCORRECT: Breaks type inference
const schema = t.Object({ name: t.String() })
app.post('/user', ({ body }) => body, { body: schema })

// ✅ ACCEPTABLE: With as const
const schema = t.Object({ name: t.String() }) as const
```

---

## 2. Plugin Architecture (CRITICAL)

### Name Plugins for Automatic Deduplication

Named plugins become singletons, preventing redundant execution:

```typescript
// ✅ CORRECT: Named plugin (singleton)
const authPlugin = new Elysia({ name: 'auth' })
  .derive(({ headers }) => ({
    user: validateToken(headers.authorization)
  }))

app.use(authPlugin)
app.group('/api', app => app.use(authPlugin))
// Auth runs only once per request

// ❌ INCORRECT: Unnamed plugin (runs multiple times)
const authPlugin = new Elysia()
  .derive(({ headers }) => ({ user: validateToken(headers.authorization) }))
```

### Use Scoped Plugins for Feature Isolation

Use scoped plugins (default) for features, global only for cross-cutting concerns:

```typescript
// ✅ CORRECT: Scoped by default
const dbPlugin = new Elysia({ name: 'db' })
  .derive({ as: 'scoped' }, () => ({ db: database }))

app
  .get('/health', () => 'OK') // No db
  .group('/api', app => app
    .use(dbPlugin) // db only in /api
    .get('/users', ({ db }) => db.users.findMany())
  )

// Global only for cross-cutting
const corsPlugin = new Elysia({ name: 'cors' })
  .onBeforeHandle({ as: 'global' }, async ({ set }) => {
    set.headers['Access-Control-Allow-Origin'] = '*'
  })
```

### Use Method Chaining for Type Inference

Every method returns a new type. Breaking the chain loses type information:

```typescript
// ✅ CORRECT: Method chaining
const app = new Elysia()
  .decorate('db', database)
  .state('version', '1.0')
  .get('/', ({ db, store }) => ({ version: store.version }))

// ❌ INCORRECT: Breaks type inference
const app = new Elysia()
app.decorate('db', database)
app.state('version', '1.0')
app.get('/', ({ db, store }) => {}) // TypeScript errors
```

---

## 3. Lifecycle Optimization (HIGH)

### Order Lifecycle Hooks Before Routes

Hooks only apply to routes registered after them:

```typescript
// ✅ CORRECT: Guard before routes
app
  .get('/public', () => 'Public')
  .guard({
    beforeHandle: ({ headers }) => validateAuth(headers)
  }, app => app
    .post('/admin', () => 'Protected')
  )

// ❌ INCORRECT: Routes before guard
app
  .post('/admin', () => 'Exposed!') // No protection
  .guard({
    beforeHandle: ({ headers }) => validateAuth(headers)
  }, app => app.get('/protected', () => 'Protected'))
```

### Use derive for Pre-Validation, resolve for Post-Validation

```typescript
app
  // derive: Before validation (raw data)
  .derive(({ headers }) => ({
    userId: extractUserId(headers.authorization)
  }))

  // Validation happens here

  // resolve: After validation (validated data)
  .resolve(async ({ userId }) => ({
    user: await getUserFromCache(userId)
  }))

  .get('/profile', ({ user }) => user)
```

### Use guard for Shared Schema and Hooks

Apply common validation and middleware to multiple routes:

```typescript
app.guard({
  headers: t.Object({
    authorization: t.String()
  }),
  beforeHandle: async ({ headers }) => {
    await validateToken(headers.authorization)
  }
}, app => app
  .get('/users', () => getUsers())
  .post('/posts', ({ body }) => createPost(body))
)
```

---

## 4. Async Patterns (HIGH)

### Use Promise.all() for Independent Operations

Eliminate waterfalls with parallel execution:

```typescript
// ❌ INCORRECT: Sequential (300ms total)
const user = await fetchUser()      // 100ms
const posts = await fetchPosts()    // 100ms
const comments = await fetchComments() // 100ms

// ✅ CORRECT: Parallel (100ms total)
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments()
])
```

### Defer await Until Data is Needed

Move await into branches or as late as possible:

```typescript
// ❌ INCORRECT: Await before early return
const user = await db.users.findUnique({ where: { id } })
if (!headers.authorization) throw new Error('Unauthorized')

// ✅ CORRECT: Early return first
if (!headers.authorization) throw new Error('Unauthorized')
const user = await db.users.findUnique({ where: { id } })
```

---

## 5. Error Handling (MEDIUM-HIGH)

### Use onError for Centralized Error Handling

```typescript
app
  .onError(({ code, error, set }) => {
    console.error(`[${code}]`, error)

    if (code === 'VALIDATION') {
      set.status = 400
      return { error: 'Validation failed', details: error.all }
    }

    set.status = 500
    return { error: 'Internal server error' }
  })

  .get('/users', () => db.users.findMany())
  .post('/posts', ({ body }) => db.posts.create({ data: body }))
```

### Provide Custom Error Messages in Schemas

```typescript
app.post('/user', ({ body }) => createUser(body), {
  body: t.Object({
    email: t.String({
      format: 'email',
      error: 'Please provide a valid email address'
    }),
    age: t.Number({
      minimum: 18,
      error: 'You must be at least 18 years old'
    })
  })
})
```

---

## 6. Response Optimization (MEDIUM)

### Use Streaming for Large Responses

Generator functions automatically enable streaming:

```typescript
app.get('/export', async function* () {
  yield 'id,name,email\n'

  let cursor = 0
  while (true) {
    const records = await db.records.findMany({ take: 1000, skip: cursor })
    if (records.length === 0) break

    for (const record of records) {
      yield `${record.id},${record.name},${record.email}\n`
    }
    cursor += 1000
  }
})
```

### Return Typed Responses for Compile-Time Optimization

Response schemas enable AOT compilation for faster serialization:

```typescript
app.get('/user/:id', ({ params }) => getUser(params.id), {
  response: t.Object({
    id: t.Number(),
    name: t.String(),
    email: t.String()
  })
})
```

---

## 7. Memory & Performance (MEDIUM)

### Use LRU Cache for Cross-Request Caching

```typescript
import { LRUCache } from 'lru-cache'

const cache = new LRUCache<number, User>({
  max: 500,
  ttl: 1000 * 60 * 5 // 5 minutes
})

app.get('/user/:id', async ({ params }) => {
  const cached = cache.get(params.id)
  if (cached) return cached

  const user = await fetchUser(params.id)
  cache.set(params.id, user)
  return user
})
```

### Leverage Bun's Native APIs

Bun's native APIs are significantly faster than Node.js equivalents:

```typescript
// ✅ CORRECT: Bun native APIs
const file = Bun.file('./data.json')
const data = await file.json()
await Bun.write('./output.json', JSON.stringify(data))
await Bun.sleep(1000)
const hash = new Bun.CryptoHasher('sha256').update(data).digest('hex')

// ❌ SLOWER: Node.js APIs
import fs from 'fs/promises'
const content = await fs.readFile('./data.json', 'utf-8')
const data = JSON.parse(content)
```

---

## 8. Advanced Patterns (LOW-MEDIUM)

### Use Macros for Reusable Schema Logic

```typescript
const plugin = new Elysia()
  .macro(({ onBeforeHandle }) => ({
    paginate(config: { maxLimit?: number }) {
      onBeforeHandle(({ query }) => {
        query.page = Math.max(1, query.page ?? 1)
        query.limit = Math.min(config.maxLimit ?? 100, query.limit ?? 10)
      })
    }
  }))

app
  .use(plugin)
  .get('/users', ({ query }) => getUsers(query.page, query.limit), {
    paginate: { maxLimit: 50 }
  })
```

### Use Trace for Performance Monitoring

```typescript
app.trace(async ({ request, onHandle }) => {
  const { time: start, end } = await onHandle
  const result = await end

  console.log(`${request.method} ${new URL(request.url).pathname}: ${result.time - start}ms`)
})
```

---

## Performance Checklist

### Critical (Apply First)
- ✅ Use TypeBox for validation
- ✅ Name all plugins
- ✅ Use method chaining
- ✅ Define schemas inline
- ✅ Scope plugins appropriately

### High Impact
- ✅ Order hooks before routes
- ✅ Use Promise.all() for parallel operations
- ✅ Defer await until needed
- ✅ Use derive/resolve correctly

### Medium Impact
- ✅ Centralize error handling
- ✅ Stream large responses
- ✅ Type response schemas
- ✅ Use LRU cache
- ✅ Leverage Bun APIs

### Advanced
- ✅ Create macros for reusable logic
- ✅ Use trace for monitoring

---

## Additional Resources

- [ElysiaJS Documentation](https://elysiajs.com)
- [TypeBox Documentation](https://github.com/sinclairzx81/typebox)
- [Bun Runtime APIs](https://bun.sh/docs/api)
- [ElysiaJS Best Practices](https://elysiajs.com/essential/best-practice)
