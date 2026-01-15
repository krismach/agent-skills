---
title: Dependency-Based Parallelization
impact: CRITICAL
impactDescription: 2-10x improvement for operations with partial dependencies
tags: async, parallelization, dependencies, better-all, performance
---

## Dependency-Based Parallelization

For operations with partial dependencies, use `better-all` to maximize parallelism. It automatically starts each task at the earliest possible moment, unlike `Promise.all` which waits for all dependencies.

**Incorrect (profile waits for config unnecessarily):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/dashboard', async (c) => {
  // All operations start in parallel, but profile could start earlier
  const [user, config] = await Promise.all([
    fetchUser(c.req.header('Authorization')),
    fetchConfig()
  ])

  // profile waits for BOTH user and config to complete
  const profile = await fetchProfile(user.id)

  return c.json({ user, config, profile })
})
```

**Correct (config and profile run in maximum parallel):**

```typescript
import { all } from 'better-all'
import { Hono } from 'hono'

const app = new Hono()

app.get('/dashboard', async (c) => {
  const auth = c.req.header('Authorization')

  const { user, config, profile } = await all({
    async user() {
      return fetchUser(auth)
    },
    async config() {
      return fetchConfig()
    },
    async profile() {
      // Starts as soon as user resolves, doesn't wait for config
      const userData = await this.$.user
      return fetchProfile(userData.id)
    }
  })

  return c.json({ user, config, profile })
})
```

**Timeline comparison:**

```
Promise.all approach:
[user fetch] -----> done
[config fetch] -> done
                     [profile fetch] ---> done
Total: fetch user + config + profile = longest

better-all approach:
[user fetch] -----> done
[config fetch] -----> done
                [profile fetch] ---> done (starts when user done)
Total: max(config, user + profile) = faster
```

**Complex example with multiple dependencies:**

```typescript
app.get('/complex-data', async (c) => {
  const userId = c.req.query('userId')

  const result = await all({
    // Independent - starts immediately
    async config() {
      return fetchConfig()
    },

    // Independent - starts immediately
    async permissions() {
      return fetchPermissions()
    },

    // Depends on config - starts when config resolves
    async settings() {
      const cfg = await this.$.config
      return fetchUserSettings(userId, cfg.settingsVersion)
    },

    // Depends on permissions - starts when permissions resolves
    async allowedActions() {
      const perms = await this.$.permissions
      return computeAllowedActions(perms)
    },

    // Depends on both settings and permissions
    async dashboard() {
      const [settings, perms] = await Promise.all([
        this.$.settings,
        this.$.permissions
      ])
      return buildDashboard(settings, perms)
    }
  })

  return c.json(result)
})
```

**Pattern: API aggregation:**

```typescript
app.get('/aggregated', async (c) => {
  const { user, posts, comments, likes } = await all({
    async user() {
      return fetchUser()
    },

    async posts() {
      const u = await this.$.user
      return fetchUserPosts(u.id)
    },

    async comments() {
      const p = await this.$.posts
      return fetchComments(p.map(post => post.id))
    },

    async likes() {
      const p = await this.$.posts
      return fetchLikes(p.map(post => post.id))
    }
  })

  return c.json({ user, posts, comments, likes })
})
```

**When to use:**
- Operations have partial dependencies (some depend on others)
- Want to maximize parallelism without manual promise orchestration
- Complex data fetching with multiple levels of dependencies
- API aggregation where some calls depend on previous results

**When NOT to use:**
- All operations are independent → use `Promise.all`
- Only sequential operations → just use `await`
- Single level of dependencies → start promises early, await late

**Installation:**

```bash
bun add better-all
```

Benefits:
- Automatic dependency resolution
- Maximum parallelism without manual orchestration
- Type-safe results
- Cleaner code than manual promise management
- 2-10x faster than naive sequential awaits

Reference: [better-all on GitHub](https://github.com/shuding/better-all)
