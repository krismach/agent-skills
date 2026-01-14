---
title: Use t.Numeric() for Numeric Strings
impact: HIGH
impactDescription: Automatic coercion with validation
tags: schema, validation, coercion, query, params
---

## Use t.Numeric() for Numeric Strings

**Impact: HIGH (Automatic coercion with validation)**

Query parameters and path parameters are always strings. Use `t.Numeric()` to accept numeric strings or numbers and automatically transform them to numbers with validation.

**Incorrect (manual parsing and validation):**

```typescript
app.get('/user/:id', ({ params }) => {
  const id = parseInt(params.id)
  if (isNaN(id) || id <= 0) {
    throw new Error('Invalid ID')
  }
  return getUserById(id)
}, {
  params: t.Object({
    id: t.String() // Type is string, requires manual parsing
  })
})
```

**Correct (using t.Numeric()):**

```typescript
app.get('/user/:id', ({ params }) => {
  // params.id is automatically a number
  return getUserById(params.id)
}, {
  params: t.Object({
    id: t.Numeric({ minimum: 1 })
  })
})
```

**Also useful for query parameters:**

```typescript
app.get('/users', ({ query }) => {
  // Both page and limit are numbers
  return getUsers(query.page, query.limit)
}, {
  query: t.Object({
    page: t.Numeric({ minimum: 1, default: 1 }),
    limit: t.Numeric({ minimum: 1, maximum: 100, default: 10 })
  })
})
```

Reference: [ElysiaJS TypeBox Patterns](https://elysiajs.com/patterns/typebox)
