---
title: Use Context Efficiently
impact: LOW-MEDIUM
impactDescription: reduces repeated parsing and object creation
tags: context, hono, patterns, efficiency
---

## Use Context Efficiently

Extract values from the Hono context once and pass them down, rather than repeatedly accessing context methods.

**Incorrect (repeated context access):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

async function validateUser(c: Context) {
  const body = await c.req.json()
  // Validation...
}

async function saveUser(c: Context) {
  const body = await c.req.json() // Parses JSON again!
  // Save...
}

async function sendEmail(c: Context) {
  const body = await c.req.json() // Parses JSON AGAIN!
  // Send email...
}

app.post('/users', async (c) => {
  await validateUser(c)
  await saveUser(c)
  await sendEmail(c)
  return c.json({ success: true })
})
```

**Correct (extract once, pass down):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

async function validateUser(data: UserData) {
  // Validation...
}

async function saveUser(data: UserData) {
  // Save...
}

async function sendEmail(email: string) {
  // Send email...
}

app.post('/users', async (c) => {
  // Extract once
  const body = await c.req.json()

  // Pass data down
  await validateUser(body)
  await saveUser(body)
  await sendEmail(body.email)

  return c.json({ success: true })
})
```

**Avoid repeated header access:**

```typescript
// Bad
app.use('*', async (c, next) => {
  logRequest(c.req.header('User-Agent'))
  await next()
  logResponse(c.req.header('User-Agent')) // Parses headers again
})

// Good
app.use('*', async (c, next) => {
  const userAgent = c.req.header('User-Agent')
  logRequest(userAgent)
  await next()
  logResponse(userAgent)
})
```

**Extract all needed values at route boundary:**

```typescript
app.get('/users/:id', async (c) => {
  // Extract all needed values once
  const id = c.req.param('id')
  const userAgent = c.req.header('User-Agent')
  const query = c.req.query()

  // Pass to business logic
  const user = await getUserById(id)
  const analytics = await trackView(id, userAgent)
  const options = parseOptions(query)

  return c.json({ user, analytics, options })
})
```

**Use context variables for sharing:**

```typescript
// Middleware extracts and stores
app.use('*', async (c, next) => {
  const user = await authenticateUser(c.req.header('Authorization'))
  c.set('user', user) // Store in context
  await next()
})

// Handler retrieves
app.get('/profile', (c) => {
  const user = c.get('user') // Type-safe retrieval
  return c.json({ user })
})
```

**Pattern: extract, validate, process:**

```typescript
app.post('/api/data', async (c) => {
  // 1. Extract (once)
  const body = await c.req.json()
  const headers = {
    contentType: c.req.header('Content-Type'),
    auth: c.req.header('Authorization'),
  }

  // 2. Validate (pure functions)
  const validation = validateData(body)
  if (!validation.success) {
    return c.json({ error: validation.error }, 400)
  }

  // 3. Process (pure functions)
  const result = await processData(validation.data, headers)

  return c.json(result)
})
```

Benefits:
- Avoid repeated JSON parsing
- Reduce header parsing overhead
- More testable (pure functions)
- Clearer data flow
- Better TypeScript inference

Reference: [Hono Context](https://hono.dev/docs/api/context)
