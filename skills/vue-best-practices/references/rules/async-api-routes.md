---
title: Parallelize API Route Dependencies
impact: HIGH
impactDescription: reduces API response time
tags: async, nuxt, api-routes, server
---

## Parallelize API Route Dependencies

In Nuxt API routes, start independent operations concurrently to minimize response time.

**Incorrect (sequential operations):**

```typescript
// server/api/dashboard.get.ts
export default defineEventHandler(async (event) => {
  const userId = getQuery(event).userId as string

  const user = await fetchUser(userId)
  const posts = await fetchPosts(userId)
  const analytics = await fetchAnalytics(userId)

  return { user, posts, analytics }
})
```

**Correct (parallel operations):**

```typescript
// server/api/dashboard.get.ts
export default defineEventHandler(async (event) => {
  const userId = getQuery(event).userId as string

  const [user, posts, analytics] = await Promise.all([
    fetchUser(userId),
    fetchPosts(userId),
    fetchAnalytics(userId)
  ])

  return { user, posts, analytics }
})
```

**With dependencies and error handling:**

```typescript
// server/api/post/[id].get.ts
export default defineEventHandler(async (event) => {
  const postId = getRouterParam(event, 'id')

  try {
    // Fetch post first
    const post = await fetchPost(postId)

    if (!post) {
      throw createError({
        statusCode: 404,
        message: 'Post not found'
      })
    }

    // Then fetch dependent data in parallel
    const [author, comments, related] = await Promise.all([
      fetchAuthor(post.authorId),
      fetchComments(post.id),
      fetchRelatedPosts(post.categoryId)
    ])

    return {
      post,
      author,
      comments,
      related
    }
  } catch (error) {
    throw createError({
      statusCode: 500,
      message: 'Failed to fetch post details'
    })
  }
})
```

Reference: [Nuxt Server Routes](https://nuxt.com/docs/guide/directory-structure/server)
