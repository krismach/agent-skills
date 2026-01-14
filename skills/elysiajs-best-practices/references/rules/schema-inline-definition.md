---
title: Define Schemas Inline for Type Inference
impact: CRITICAL
impactDescription: Enables automatic type inference
tags: schema, typescript, types, inference
---

## Define Schemas Inline for Type Inference

**Impact: CRITICAL (Enables automatic type inference)**

Elysia requires schemas to be defined inline or as const assertions to properly infer TypeScript types. Extracting schemas to variables without `as const` breaks type inference.

**Incorrect (extracted schema without const):**

```typescript
// Type inference is lost
const userSchema = t.Object({
  name: t.String(),
  age: t.Number()
})

app.post('/user', ({ body }) => {
  // body type is unknown
  return body
}, {
  body: userSchema
})
```

**Correct (inline schema definition):**

```typescript
app.post('/user', ({ body }) => {
  // body is properly typed as { name: string; age: number }
  return body
}, {
  body: t.Object({
    name: t.String(),
    age: t.Number()
  })
})
```

**Acceptable (const assertion for reusable schemas):**

```typescript
const userSchema = t.Object({
  name: t.String(),
  age: t.Number()
}) as const

app.post('/user', ({ body }) => {
  // Type inference works with as const
  return body
}, {
  body: userSchema
})
```

**Best practice for shared schemas:**

```typescript
// Create a model namespace
const UserModel = {
  create: t.Object({
    name: t.String(),
    age: t.Number()
  }),
  update: t.Object({
    name: t.Optional(t.String()),
    age: t.Optional(t.Number())
  })
} as const

app.post('/user', ({ body }) => body, { body: UserModel.create })
app.patch('/user/:id', ({ body }) => body, { body: UserModel.update })
```

Reference: [ElysiaJS TypeScript Patterns](https://elysiajs.com/patterns/typescript)
