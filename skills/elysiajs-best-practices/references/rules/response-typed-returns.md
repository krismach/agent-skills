---
title: Return Typed Responses for Compile-Time Optimization
impact: MEDIUM
impactDescription: Enables ahead-of-time compilation
tags: response, performance, typescript, compilation
---

## Return Typed Responses for Compile-Time Optimization

**Impact: MEDIUM (Enables ahead-of-time compilation)**

Define response schemas to enable ElysiaJS's Sucrose compiler to optimize response handling at compile time, generating faster serialization code.

**Incorrect (untyped responses):**

```typescript
app.get('/user/:id', async ({ params }) => {
  const user = await db.users.findUnique({ where: { id: params.id } })
  // No response schema - runtime serialization
  return user
})
```

**Correct (typed response schema):**

```typescript
app.get('/user/:id', async ({ params }) => {
  const user = await db.users.findUnique({ where: { id: params.id } })
  return user
}, {
  params: t.Object({
    id: t.Numeric()
  }),
  response: t.Object({
    id: t.Number(),
    name: t.String(),
    email: t.String(),
    createdAt: t.String()
  })
})
// Elysia compiles optimized serialization code
```

**Pattern: Multiple response types:**

```typescript
app.get('/search', ({ query }) => {
  if (!query.q) {
    return { error: 'Query required' }
  }

  const results = search(query.q)
  return { results }
}, {
  query: t.Object({
    q: t.Optional(t.String())
  }),
  response: {
    200: t.Object({
      results: t.Array(t.Object({
        id: t.Number(),
        title: t.String()
      }))
    }),
    400: t.Object({
      error: t.String()
    })
  }
})
```

**Reusable response models:**

```typescript
const UserResponse = t.Object({
  id: t.Number(),
  name: t.String(),
  email: t.String(),
  role: t.Union([t.Literal('admin'), t.Literal('user')])
})

const ListResponse = <T extends TSchema>(item: T) => t.Object({
  data: t.Array(item),
  total: t.Number(),
  page: t.Number()
})

app
  .get('/user/:id', ({ params }) => getUser(params.id), {
    response: UserResponse
  })

  .get('/users', ({ query }) => getUsers(query), {
    response: ListResponse(UserResponse)
  })
```

**Error responses:**

```typescript
const ErrorResponse = t.Object({
  error: t.String(),
  code: t.String(),
  details: t.Optional(t.Any())
})

const SuccessResponse = <T extends TSchema>(data: T) => t.Object({
  success: t.Literal(true),
  data
})

app
  .get('/api/posts', () => ({ success: true, data: getPosts() }), {
    response: {
      200: SuccessResponse(t.Array(t.Object({
        id: t.Number(),
        title: t.String()
      }))),
      500: ErrorResponse
    }
  })
```

**Response validation and stripping:**

```typescript
app.get('/user/:id', async ({ params }) => {
  // Query returns more fields than needed
  const user = await db.users.findUnique({
    where: { id: params.id }
  })

  // Response schema automatically strips extra fields
  return user
}, {
  response: t.Object({
    id: t.Number(),
    name: t.String(),
    email: t.String()
    // passwordHash, createdAt, etc. are automatically removed
  })
})
```

**Status code specific responses:**

```typescript
app.post('/user', ({ body, set }) => {
  const existing = findUser(body.email)

  if (existing) {
    set.status = 409
    return {
      error: 'User already exists',
      code: 'DUPLICATE_USER'
    }
  }

  const user = createUser(body)
  set.status = 201
  return user
}, {
  response: {
    201: t.Object({
      id: t.Number(),
      name: t.String(),
      email: t.String()
    }),
    409: t.Object({
      error: t.String(),
      code: t.String()
    })
  }
})
```

**Guard with shared response schema:**

```typescript
app.guard({
  response: t.Object({
    success: t.Boolean(),
    data: t.Any(),
    timestamp: t.Number()
  })
}, app => app
  .get('/users', () => ({
    success: true,
    data: getUsers(),
    timestamp: Date.now()
  }))

  .get('/posts', () => ({
    success: true,
    data: getPosts(),
    timestamp: Date.now()
  }))
)
```

Reference: [ElysiaJS Response Schema](https://elysiajs.com/essential/validation#response)
