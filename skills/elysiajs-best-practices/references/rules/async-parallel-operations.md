---
title: Use Promise.all() for Independent Operations
impact: CRITICAL
impactDescription: 2-10× improvement
tags: async, parallelization, promises, performance
---

## Use Promise.all() for Independent Operations

**Impact: CRITICAL (2-10× improvement)**

When async operations have no interdependencies, execute them concurrently using `Promise.all()` or `Promise.allSettled()` to eliminate sequential waterfalls.

**Incorrect (sequential execution):**

```typescript
app.get('/dashboard', async () => {
  // 3 round trips = 300ms+ total
  const user = await fetchUser()      // 100ms
  const posts = await fetchPosts()    // 100ms
  const comments = await fetchComments() // 100ms

  return { user, posts, comments }
})
```

**Correct (parallel execution):**

```typescript
app.get('/dashboard', async () => {
  // 1 round trip = ~100ms total
  const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments()
  ])

  return { user, posts, comments }
})
```

**With error handling using Promise.allSettled():**

```typescript
app.get('/dashboard', async () => {
  const results = await Promise.allSettled([
    fetchUser(),
    fetchPosts(),
    fetchComments()
  ])

  return {
    user: results[0].status === 'fulfilled' ? results[0].value : null,
    posts: results[1].status === 'fulfilled' ? results[1].value : [],
    comments: results[2].status === 'fulfilled' ? results[2].value : []
  }
})
```

**Pattern: Start promises early, await late:**

```typescript
app.get('/user/:id', async ({ params }) => {
  // Start all promises immediately
  const userPromise = db.users.findUnique({ where: { id: params.id } })
  const postsPromise = db.posts.findMany({ where: { userId: params.id } })
  const statsPromise = db.stats.findUnique({ where: { userId: params.id } })

  // Await only when needed
  const user = await userPromise
  if (!user) throw new Error('User not found')

  // Continue with other promises
  const [posts, stats] = await Promise.all([postsPromise, statsPromise])

  return { user, posts, stats }
})
```

**Parallel with dependencies:**

```typescript
app.get('/report', async () => {
  // Step 1: Fetch user and config in parallel
  const [user, config] = await Promise.all([
    fetchUser(),
    fetchConfig()
  ])

  // Step 2: Use results to fetch related data in parallel
  const [posts, analytics] = await Promise.all([
    fetchUserPosts(user.id),
    fetchAnalytics(user.id, config.dateRange)
  ])

  return { user, posts, analytics }
})
```

**Using with Elysia resolve:**

```typescript
app
  .resolve(async ({ params }) => {
    // Parallel data fetching in resolve hook
    const [user, permissions, settings] = await Promise.all([
      db.users.findUnique({ where: { id: params.userId } }),
      db.permissions.findMany({ where: { userId: params.userId } }),
      db.settings.findUnique({ where: { userId: params.userId } })
    ])

    return { user, permissions, settings }
  })
  .get('/user/:userId', ({ user, permissions, settings }) => ({
    user,
    permissions,
    settings
  }), {
    params: t.Object({
      userId: t.Numeric()
    })
  })
```

Reference: [MDN Promise.all()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
