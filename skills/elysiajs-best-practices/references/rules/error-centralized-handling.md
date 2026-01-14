---
title: Use onError for Centralized Error Handling
impact: MEDIUM-HIGH
impactDescription: Consistent error responses and logging
tags: error, lifecycle, logging, debugging
---

## Use onError for Centralized Error Handling

**Impact: MEDIUM-HIGH (Consistent error responses and logging)**

Use the `onError` lifecycle hook to handle all errors in one place, ensuring consistent error responses, proper logging, and preventing information leakage.

**Incorrect (scattered error handling):**

```typescript
app
  .get('/users', async () => {
    try {
      return await db.users.findMany()
    } catch (error) {
      console.error('Error fetching users:', error)
      return { error: 'Failed to fetch users' }
    }
  })

  .post('/posts', async ({ body }) => {
    try {
      return await db.posts.create({ data: body })
    } catch (error) {
      console.error('Error creating post:', error)
      return { error: 'Failed to create post' }
    }
  })
// Inconsistent error handling and responses
```

**Correct (centralized with onError):**

```typescript
app
  .onError(({ code, error, set }) => {
    console.error(`[${code}]`, error)

    if (code === 'VALIDATION') {
      set.status = 400
      return {
        error: 'Validation failed',
        details: error.all
      }
    }

    if (code === 'NOT_FOUND') {
      set.status = 404
      return { error: 'Resource not found' }
    }

    // Don't leak internal errors
    set.status = 500
    return {
      error: 'Internal server error',
      requestId: crypto.randomUUID()
    }
  })

  .get('/users', () => db.users.findMany())
  .post('/posts', ({ body }) => db.posts.create({ data: body }))
// All errors handled consistently
```

**Pattern: Custom error classes:**

```typescript
class NotFoundError extends Error {
  constructor(resource: string) {
    super(`${resource} not found`)
    this.name = 'NotFoundError'
  }
}

class UnauthorizedError extends Error {
  constructor(message = 'Unauthorized') {
    super(message)
    this.name = 'UnauthorizedError'
  }
}

app
  .onError(({ error, set }) => {
    if (error instanceof NotFoundError) {
      set.status = 404
      return { error: error.message }
    }

    if (error instanceof UnauthorizedError) {
      set.status = 401
      return { error: error.message }
    }

    if (error instanceof Error) {
      set.status = 500
      return { error: 'Internal server error' }
    }
  })

  .get('/user/:id', async ({ params }) => {
    const user = await db.users.findUnique({ where: { id: params.id } })
    if (!user) throw new NotFoundError('User')
    return user
  })
```

**Validation error handling:**

```typescript
app
  .onError(({ code, error, set }) => {
    if (code === 'VALIDATION') {
      set.status = 400

      // Access TypeBox validation errors
      const errors = error.all.map(err => ({
        field: err.path,
        message: err.message,
        expected: err.schema.type
      }))

      return {
        error: 'Validation failed',
        errors
      }
    }
  })

  .post('/user', ({ body }) => createUser(body), {
    body: t.Object({
      name: t.String({ minLength: 3 }),
      email: t.String({ format: 'email' }),
      age: t.Number({ minimum: 0, maximum: 120 })
    })
  })
```

**Structured logging with error context:**

```typescript
app
  .derive(({ request }) => ({
    requestId: crypto.randomUUID(),
    timestamp: Date.now()
  }))

  .onError(({ code, error, requestId, timestamp, path, request }) => {
    // Structured error logging
    logger.error({
      requestId,
      timestamp,
      path,
      method: request.method,
      code,
      error: error.message,
      stack: error.stack
    })

    return {
      error: 'An error occurred',
      requestId
    }
  })
```

**Scoped error handlers:**

```typescript
app
  // Global error handler
  .onError({ as: 'global' }, ({ error, set }) => {
    set.status = 500
    return { error: 'Internal server error' }
  })

  // API-specific error handler
  .group('/api', app => app
    .onError(({ error, set }) => {
      // API-specific error formatting
      set.status = error.status ?? 500
      return {
        success: false,
        error: error.message,
        code: error.code
      }
    })
    .get('/users', () => getUsers())
  )
```

Reference: [ElysiaJS Error Handling](https://elysiajs.com/patterns/error-handling)
