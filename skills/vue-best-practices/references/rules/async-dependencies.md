---
title: Start Promises Early, Await Late
impact: HIGH
impactDescription: reduces waterfall latency
tags: async, promises, parallelization
---

## Start Promises Early, Await Late

When operations have partial dependencies, start all promises immediately but only await when you need their results.

**Incorrect (sequential waterfall):**

```typescript
export async function usePostWithAuthor(postId: string) {
  const post = await fetchPost(postId)
  const author = await fetchAuthor(post.authorId) // Waterfall!

  return { post, author }
}
```

**Correct (parallel start, sequential await):**

```typescript
export async function usePostWithAuthor(postId: string) {
  const postPromise = fetchPost(postId)
  const post = await postPromise

  // Both requests in flight simultaneously
  const authorPromise = fetchAuthor(post.authorId)
  const author = await authorPromise

  return { post, author }
}
```

**Even better (with multiple dependencies):**

```typescript
export async function usePostDetails(postId: string) {
  const postPromise = fetchPost(postId)
  const post = await postPromise

  // Start all dependent fetches immediately
  const [author, comments, relatedPosts] = await Promise.all([
    fetchAuthor(post.authorId),
    fetchComments(post.id),
    fetchRelatedPosts(post.categoryId)
  ])

  return { post, author, comments, relatedPosts }
}
```

**Use Promise.allSettled() for partial failures:**

```typescript
export async function useUserDashboard(userId: string) {
  const userPromise = fetchUser(userId)
  const user = await userPromise

  // Some requests might fail, but we want the others to succeed
  const results = await Promise.allSettled([
    fetchPosts(user.id),
    fetchFollowers(user.id),
    fetchAnalytics(user.id) // Might fail due to permissions
  ])

  return {
    user,
    posts: results[0].status === 'fulfilled' ? results[0].value : [],
    followers: results[1].status === 'fulfilled' ? results[1].value : [],
    analytics: results[2].status === 'fulfilled' ? results[2].value : null
  }
}
```

Reference: [MDN Promise.allSettled](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
