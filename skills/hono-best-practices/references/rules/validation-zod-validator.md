---
title: Use Zod Validator for Type Safety
impact: MEDIUM-HIGH
impactDescription: runtime validation with zero overhead TypeScript types
tags: validation, zod, typescript, type-safety
---

## Use Zod Validator for Type Safety

Use Zod with Hono's validator middleware for runtime type safety that generates TypeScript types automatically.

**Incorrect (manual validation, no type safety):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.post('/users', async (c) => {
  const body = await c.req.json()

  // Manual validation - error-prone
  if (!body.name || typeof body.name !== 'string') {
    return c.json({ error: 'Invalid name' }, 400)
  }
  if (!body.email || typeof body.email !== 'string') {
    return c.json({ error: 'Invalid email' }, 400)
  }
  if (!body.age || typeof body.age !== 'number') {
    return c.json({ error: 'Invalid age' }, 400)
  }

  // body is still 'any' type
  const name = body.name
  const email = body.email
  const age = body.age
})
```

**Correct (Zod validator with automatic types):**

```typescript
import { Hono } from 'hono'
import { z } from 'zod'
import { zValidator } from '@hono/zod-validator'

const app = new Hono()

// Define schema once
const userSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(120),
})

app.post('/users', zValidator('json', userSchema), async (c) => {
  // Fully typed and validated!
  const user = c.req.valid('json')
  // user.name is string
  // user.email is string (valid email)
  // user.age is number (0-120)

  return c.json({ success: true })
})
```

**Validating different inputs:**

```typescript
import { zValidator } from '@hono/zod-validator'

// Query parameters
const searchSchema = z.object({
  q: z.string(),
  page: z.string().transform(Number).pipe(z.number().int().min(1)),
  limit: z.string().transform(Number).pipe(z.number().int().max(100)).optional(),
})

app.get('/search', zValidator('query', searchSchema), (c) => {
  const { q, page, limit } = c.req.valid('query')
  // All properly typed!
})

// URL parameters
const idSchema = z.object({
  id: z.string().uuid(),
})

app.get('/users/:id', zValidator('param', idSchema), (c) => {
  const { id } = c.req.valid('param')
  // id is a valid UUID string
})

// Headers
const authSchema = z.object({
  authorization: z.string().regex(/^Bearer .+$/),
})

app.use('*', zValidator('header', authSchema), (c, next) => {
  const { authorization } = c.req.valid('header')
  // authorization is properly formatted
  return next()
})
```

**Cache schemas for reuse:**

```typescript
// Bad - creates new schema each request
app.post('/users', zValidator('json', z.object({
  name: z.string()
})), handler)

// Good - reuse schema instance
const userSchema = z.object({ name: z.string() })
app.post('/users', zValidator('json', userSchema), handler)
```

**Custom error handling:**

```typescript
import { zValidator } from '@hono/zod-validator'

app.post(
  '/users',
  zValidator('json', userSchema, (result, c) => {
    if (!result.success) {
      return c.json({
        error: 'Validation failed',
        details: result.error.flatten(),
      }, 400)
    }
  }),
  async (c) => {
    const user = c.req.valid('json')
    return c.json({ success: true })
  }
)
```

**Complex validation with refinements:**

```typescript
const passwordSchema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword'],
})
```

Benefits:
- Runtime validation catches invalid data
- Automatic TypeScript types (no manual typing)
- Better error messages
- Composable schemas
- Transform and coerce values
- Works with RPC for end-to-end type safety

Reference: [Hono Validation](https://hono.dev/docs/guides/validation)
