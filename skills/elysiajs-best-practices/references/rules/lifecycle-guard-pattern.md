---
title: Use guard for Shared Schema and Hooks
impact: MEDIUM-HIGH
impactDescription: Reduces code duplication
tags: lifecycle, guard, schema, middleware
---

## Use guard for Shared Schema and Hooks

**Impact: MEDIUM-HIGH (Reduces code duplication)**

Use `guard` to apply the same schema and hooks to multiple routes at once, eliminating repetitive middleware and validation logic.

**Incorrect (repeated validation and hooks):**

```typescript
app
  .get('/users', ({ headers }) => {
    if (!headers.authorization) throw new Error('Unauthorized')
    return getUsers()
  }, {
    headers: t.Object({
      authorization: t.String()
    })
  })

  .post('/posts', ({ headers, body }) => {
    if (!headers.authorization) throw new Error('Unauthorized')
    return createPost(body)
  }, {
    headers: t.Object({
      authorization: t.String()
    }),
    body: t.Object({
      title: t.String(),
      content: t.String()
    })
  })
// Repeated auth logic and header schema
```

**Correct (using guard):**

```typescript
app.guard({
  headers: t.Object({
    authorization: t.String()
  }),
  beforeHandle: async ({ headers }) => {
    const user = await validateToken(headers.authorization)
    if (!user) throw new Error('Unauthorized')
  }
}, app => app
  .get('/users', () => getUsers())
  .post('/posts', ({ body }) => createPost(body), {
    body: t.Object({
      title: t.String(),
      content: t.String()
    })
  })
)
```

**Pattern: Nested guards for progressive authorization:**

```typescript
app
  .guard({
    // All routes require authentication
    headers: t.Object({
      authorization: t.String()
    })
  }, app => app
    .derive(async ({ headers }) => ({
      user: await validateToken(headers.authorization)
    }))
    .onBeforeHandle(({ user }) => {
      if (!user) throw new Error('Unauthorized')
    })

    // Public authenticated routes
    .get('/profile', ({ user }) => user)

    // Admin routes require additional checks
    .guard({
      beforeHandle: ({ user }) => {
        if (!user.isAdmin) throw new Error('Forbidden')
      }
    }, app => app
      .get('/admin/users', () => getAllUsers())
      .delete('/admin/users/:id', ({ params }) => deleteUser(params.id))
    )
  )
```

**Guard with scoped plugins:**

```typescript
const authGuard = new Elysia({ name: 'auth-guard' })
  .guard({
    as: 'scoped', // Only applies to parent + children
    headers: t.Object({
      authorization: t.String()
    })
  }, app => app
    .derive(async ({ headers }) => ({
      user: await validateToken(headers.authorization)
    }))
  )

app
  .get('/public', () => 'Public') // No auth
  .use(authGuard)
  .get('/protected', ({ user }) => user) // Has auth
```

**Guard with response schema:**

```typescript
app.guard({
  response: t.Object({
    success: t.Boolean(),
    data: t.Any()
  })
}, app => app
  .get('/users', () => ({
    success: true,
    data: getUsers()
  }))
  .get('/posts', () => ({
    success: true,
    data: getPosts()
  }))
  // All responses are validated against the schema
)
```

Reference: [ElysiaJS Guard Pattern](https://elysiajs.com/essential/life-cycle#guard)
