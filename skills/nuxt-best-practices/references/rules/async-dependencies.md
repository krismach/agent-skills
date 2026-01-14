---
title: Dependency-Based Parallelization
impact: CRITICAL
impactDescription: 2-10× improvement for complex async workflows
tags: async, parallelization, dependencies, better-all
---

## Dependency-Based Parallelization

**Impact: CRITICAL (2-10× improvement)**

For operations with partial dependencies, use `better-all` to maximize parallelism. It automatically starts each task at the earliest possible moment.

**Incorrect: profile waits for config unnecessarily**

```typescript
// server/api/dashboard.get.ts
export default defineEventHandler(async (event) => {
  const [user, config] = await Promise.all([
    fetchUser(event),
    fetchConfig()
  ])
  const profile = await fetchProfile(user.id)

  return { user, config, profile }
})
// If fetchUser takes 1s, fetchConfig 10s, fetchProfile 10s
// Total: 11s (config blocks profile unnecessarily)
```

**Correct: config and profile run in parallel**

```typescript
// server/api/dashboard.get.ts
import { all } from 'better-all'

export default defineEventHandler(async (event) => {
  const { user, config, profile } = await all({
    async user() { return fetchUser(event) },
    async config() { return fetchConfig() },
    async profile() {
      return fetchProfile((await this.$.user).id)
    }
  })

  return { user, config, profile }
})
// Total: 11s (config and profile run in parallel after user)
```

**Benefits:**
- `config` starts immediately (no dependencies)
- `user` starts immediately (no dependencies)
- `profile` starts as soon as `user` resolves (doesn't wait for `config`)
- All independent work parallelizes automatically

**Complex example with multiple dependencies:**

```typescript
// Composable with complex async logic
export const useComplexData = async () => {
  const { auth, settings, userData, permissions, posts, analytics } = await all({
    async auth() {
      return await $fetch('/api/auth/session')
    },
    async settings() {
      return await $fetch('/api/settings')
    },
    async userData() {
      const auth = await this.$.auth
      return await $fetch(`/api/users/${auth.userId}`)
    },
    async permissions() {
      const auth = await this.$.auth
      return await $fetch(`/api/permissions/${auth.userId}`)
    },
    async posts() {
      const userData = await this.$.userData
      return await $fetch('/api/posts', {
        query: { authorId: userData.id }
      })
    },
    async analytics() {
      const [userData, posts] = await Promise.all([
        this.$.userData,
        this.$.posts
      ])
      return await $fetch('/api/analytics', {
        method: 'POST',
        body: { userId: userData.id, postIds: posts.map(p => p.id) }
      })
    }
  })

  return { auth, settings, userData, permissions, posts, analytics }
}
```

**Execution timeline:**
```
Time 0ms:   auth, settings start in parallel
Time 100ms: auth completes → userData, permissions start in parallel
Time 200ms: userData completes → posts starts
Time 500ms: posts completes → analytics starts (with userData + posts)
Time 800ms: all complete
```

**Without better-all (sequential):**
```
auth → userData → posts → analytics = 500ms total
(settings and permissions ignored in this path)
```

**With manual Promise.all (suboptimal):**
```
Promise.all([auth, settings]) → Promise.all([userData, permissions]) → posts → analytics
Still slower because posts waits for permissions unnecessarily
```

**Installation:**

```bash
npm install better-all
```

**Type safety:**

```typescript
const result = await all({
  async user() { return { id: '123', name: 'John' } },
  async posts() {
    const user = await this.$.user
    // user is automatically typed as { id: string, name: string }
    return fetchPosts(user.id)
  }
})

result.user // { id: string, name: string }
result.posts // Post[]
```

**Error handling:**

Use `all()` for fail-fast behavior (like Promise.all):

```typescript
const result = await all({
  async a() { throw new Error('Failed') },
  async b() { return 'success' }
})
// Throws error from task 'a'
```

Use `allSettled()` for graceful degradation:

```typescript
const result = await allSettled({
  async a() { throw new Error('Failed') },
  async b() { return 'success' }
})

result.a // { status: 'rejected', reason: Error }
result.b // { status: 'fulfilled', value: 'success' }
```

**When to use better-all:**
- Multiple async operations with partial dependencies
- Complex server-side data fetching
- API routes that aggregate data from multiple sources
- Composables that orchestrate multiple API calls

**When NOT to use:**
- Simple independent operations (use Promise.all)
- Single sequential chain (use await)
- Only 2-3 operations (manual optimization is fine)

Reference: [https://github.com/shuding/better-all](https://github.com/shuding/better-all)
