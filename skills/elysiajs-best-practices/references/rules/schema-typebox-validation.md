---
title: Use TypeBox for Schema Validation
impact: CRITICAL
impactDescription: 18× faster validation than Zod
tags: schema, validation, typebox, performance
---

## Use TypeBox for Schema Validation

**Impact: CRITICAL (18× faster validation than Zod)**

ElysiaJS uses TypeBox (Elysia.t) with AOT compilation in Bun for significantly faster validation. A single TypeBox schema provides runtime validation, type coercion, TypeScript types, and OpenAPI documentation, eliminating schema duplication.

**Incorrect (using external validation libraries):**

```typescript
import { z } from 'zod'

app.post('/user', async ({ body }) => {
  // Manual validation - slower, no auto type inference
  const userSchema = z.object({
    name: z.string(),
    age: z.number()
  })
  const validated = userSchema.parse(body)
  return validated
})
```

**Correct (using Elysia.t TypeBox):**

```typescript
app.post('/user', ({ body }) => {
  return body
}, {
  body: t.Object({
    name: t.String(),
    age: t.Number()
  })
  // Automatic validation, type inference, and OpenAPI schema
})
```

**Advanced: Custom error messages:**

```typescript
app.post('/user', ({ body }) => body, {
  body: t.Object({
    name: t.String({
      minLength: 3,
      error: 'Name must be at least 3 characters'
    }),
    age: t.Number({
      minimum: 0,
      maximum: 120,
      error: 'Age must be between 0 and 120'
    })
  })
})
```

Reference: [ElysiaJS Validation](https://elysiajs.com/essential/validation)
