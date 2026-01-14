---
title: Defer await Until Data is Needed
impact: HIGH
impactDescription: Reduces blocking time
tags: async, await, performance, lazy-evaluation
---

## Defer await Until Data is Needed

**Impact: HIGH (Reduces blocking time)**

Don't await promises until you actually need their values. Move await statements into branches or as late as possible to allow parallel execution and early returns.

**Incorrect (blocking unnecessarily):**

```typescript
app.post('/user', async ({ body, headers }) => {
  // Always waits for user lookup, even if auth fails
  const existingUser = await db.users.findUnique({
    where: { email: body.email }
  })

  // Check auth after slow DB query
  if (!headers.authorization) {
    throw new Error('Unauthorized')
  }

  if (existingUser) {
    throw new Error('User already exists')
  }

  return createUser(body)
})
```

**Correct (defer await to branch):**

```typescript
app.post('/user', async ({ body, headers }) => {
  // Fast auth check first
  if (!headers.authorization) {
    throw new Error('Unauthorized')
  }

  // Start promise but don't await yet
  const existingUserPromise = db.users.findUnique({
    where: { email: body.email }
  })

  // Do other work...
  const hashedPassword = await hashPassword(body.password)

  // Await only when needed
  const existingUser = await existingUserPromise
  if (existingUser) {
    throw new Error('User already exists')
  }

  return createUser({ ...body, password: hashedPassword })
})
```

**Pattern: Conditional data fetching:**

```typescript
app.get('/posts', async ({ query }) => {
  const postsPromise = db.posts.findMany({
    take: query.limit,
    skip: query.offset
  })

  // Only fetch total count if needed
  if (query.includeCount) {
    const [posts, total] = await Promise.all([
      postsPromise,
      db.posts.count()
    ])
    return { posts, total }
  }

  // Otherwise just return posts
  return { posts: await postsPromise }
}, {
  query: t.Object({
    limit: t.Numeric({ default: 10 }),
    offset: t.Numeric({ default: 0 }),
    includeCount: t.Optional(t.Boolean())
  })
})
```

**Early return optimization:**

```typescript
app.get('/user/:id/profile', async ({ params }) => {
  // Start fetching user
  const userPromise = db.users.findUnique({
    where: { id: params.id }
  })

  // Check cache first (fast)
  const cached = await getCachedProfile(params.id)
  if (cached) {
    return cached // Early return, user query may still be running
  }

  // Await only if we need to fetch fresh data
  const user = await userPromise
  if (!user) throw new Error('Not found')

  const profile = await buildProfile(user)
  await cacheProfile(params.id, profile)
  return profile
})
```

**Move await into branches:**

```typescript
app.post('/process', async ({ body }) => {
  // Don't await here
  const dataPromise = fetchData(body.id)

  if (body.type === 'quick') {
    // Quick path doesn't need the data
    return { status: 'queued' }
  }

  if (body.type === 'validate') {
    // Only await in this branch
    const data = await dataPromise
    return validateData(data)
  }

  // Full processing path
  const data = await dataPromise
  return processData(data)
}, {
  body: t.Object({
    id: t.String(),
    type: t.Union([
      t.Literal('quick'),
      t.Literal('validate'),
      t.Literal('full')
    ])
  })
})
```

Reference: [JavaScript Async Best Practices](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
