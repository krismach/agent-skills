# Dependency-Based Parallelization

## Section

Async & Request Lifecycle

## Summary

For operations with partial dependencies, use `better-all` to maximize parallelism. It automatically starts each task at the earliest possible moment, reducing total request time.

## Incorrect

```javascript
// Sequential: user, then config, then posts and profile
fastify.get('/dashboard/:userId', async (request, reply) => {
  const user = await getUser(request.params.userId)
  const config = await getAppConfig()
  const posts = await getUserPosts(user.id)
  const profile = await getUserProfile(user.id)

  return { user, config, posts, profile }
})

// Promise.all: config waits unnecessarily before posts/profile start
fastify.get('/dashboard/:userId', async (request, reply) => {
  const [user, config] = await Promise.all([
    getUser(request.params.userId),
    getAppConfig()
  ])

  const [posts, profile] = await Promise.all([
    getUserPosts(user.id),
    getUserProfile(user.id)
  ])

  return { user, config, posts, profile }
})
```

In the Promise.all example, `posts` and `profile` wait for both `user` AND `config` to complete, even though they only need `user`.

## Correct

```javascript
import { all } from 'better-all'

fastify.get('/dashboard/:userId', async (request, reply) => {
  const { user, config, posts, profile } = await all({
    async user() {
      return getUser(request.params.userId)
    },
    async config() {
      return getAppConfig()
    },
    async posts() {
      const userData = await this.$.user
      return getUserPosts(userData.id)
    },
    async profile() {
      const userData = await this.$.user
      return getUserProfile(userData.id)
    }
  })

  return { user, config, posts, profile }
})
```

With `better-all`:
- `user` and `config` start immediately (parallel)
- `posts` and `profile` start as soon as `user` completes
- `posts` and `profile` don't wait for `config` to finish

## Real-World Example

```javascript
import { all } from 'better-all'

// Complex route with multiple dependencies
fastify.get('/project/:id', async (request, reply) => {
  const { session, project, permissions, analytics, team, activity } = await all({
    // Independent: starts immediately
    async session() {
      return getSession(request.headers.authorization)
    },

    // Independent: starts immediately
    async project() {
      return getProject(request.params.id)
    },

    // Depends on session: starts when session completes
    async permissions() {
      const s = await this.$.session
      return getUserPermissions(s.userId, request.params.id)
    },

    // Depends on project: starts when project completes
    async analytics() {
      const p = await this.$.project
      return getProjectAnalytics(p.id)
    },

    // Depends on project: starts when project completes (parallel with analytics)
    async team() {
      const p = await this.$.project
      return getProjectTeam(p.teamId)
    },

    // Depends on both session and project: starts when both complete
    async activity() {
      const [s, p] = await Promise.all([this.$.session, this.$.project])
      return getUserActivityOnProject(s.userId, p.id)
    }
  })

  return { project, permissions, analytics, team, activity }
})
```

**Execution timeline:**
```
Time 0ms:   session, project start (parallel)
Time 50ms:  session completes → permissions starts
Time 100ms: project completes → analytics, team start (parallel)
Time 100ms: both complete → activity starts
Time 150ms: all complete
```

Compare to sequential (500ms+) or naive Promise.all (200ms+).

## Installation

```bash
npm install better-all
```

## Why

This pattern is especially valuable in Fastify because:

1. **Reduced latency** - Eliminates unnecessary waiting between dependent operations
2. **Cleaner code** - Dependencies are explicit and self-documenting
3. **Better than manual optimization** - No need to carefully orchestrate Promise.all chains
4. **Scales with complexity** - More dependencies = bigger performance gain

**Performance impact:** 2-10× improvement over sequential, 1.5-3× improvement over manual Promise.all chains.

## When to Use

Use `better-all` when:
- You have 3+ async operations in a route handler
- Some operations depend on results from others
- Operations have partial dependencies (not all depend on all)

Don't use when:
- All operations are independent (use Promise.all)
- Operations are fully sequential (await normally)
- Only 2 operations total (manual optimization is simpler)

Reference: [https://github.com/shuding/better-all](https://github.com/shuding/better-all)
