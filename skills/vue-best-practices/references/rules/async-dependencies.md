---
title: Dependency-Based Parallelization
impact: CRITICAL
impactDescription: 2-10× improvement
tags: async, promises, parallelization, dependencies, better-all
---

## Dependency-Based Parallelization

When operations have partial dependencies, maximize parallelism by starting all promises at the earliest possible moment.

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

**Best: Automatic parallelization with better-all:**

For complex dependency chains, use `better-all` to automatically maximize parallelism:

```typescript
import { all } from 'better-all'

// Incorrect: Manual orchestration, config waits for profile unnecessarily
export async function usePostDetails(postId: string) {
  const [post, config] = await Promise.all([
    fetchPost(postId),
    fetchConfig()
  ])
  const profile = await fetchProfile(post.authorId) // Config blocked by this

  return { post, config, profile }
}

// Correct: better-all automatically parallelizes config with profile
export async function usePostDetails(postId: string) {
  const { post, config, profile } = await all({
    async post() {
      return fetchPost(postId)
    },
    async config() {
      return fetchConfig() // Runs immediately, parallel with post
    },
    async profile() {
      // Waits only for post, runs parallel with config
      return fetchProfile((await this.$.post).authorId)
    }
  })

  return { post, config, profile }
}
```

**In Vue composables:**

```typescript
// composables/useUserContent.ts
import { all } from 'better-all'

export async function useUserContent(userId: string) {
  const data = await all({
    async user() {
      return fetchUser(userId)
    },
    async settings() {
      return fetchSettings(userId) // Parallel with user
    },
    async posts() {
      const user = await this.$.user
      return fetchPosts(user.id) // Waits only for user
    },
    async followers() {
      const user = await this.$.user
      return fetchFollowers(user.id) // Parallel with posts
    }
  })

  return data
}
```

**With Nuxt 3 server routes:**

```typescript
// server/api/dashboard/[userId].get.ts
import { all } from 'better-all'

export default defineEventHandler(async (event) => {
  const userId = getRouterParam(event, 'userId')!

  const dashboard = await all({
    async user() {
      return fetchUser(userId)
    },
    async systemConfig() {
      return fetchConfig() // Independent, runs immediately
    },
    async profile() {
      const user = await this.$.user
      return fetchProfile(user.profileId)
    },
    async stats() {
      const user = await this.$.user
      return fetchStats(user.id) // Parallel with profile
    }
  })

  return dashboard
})
```

Benefits of `better-all`:
- Automatic dependency resolution
- Maximal parallelization without manual orchestration
- Type-safe with full inference
- Works with any JavaScript async code
- Small bundle size (~1KB)

Reference: [better-all](https://github.com/shuding/better-all) | [MDN Promise.allSettled](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
