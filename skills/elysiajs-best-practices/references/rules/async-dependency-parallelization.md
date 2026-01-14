---
title: Use better-all for Dependency-Based Parallelization
impact: CRITICAL
impactDescription: 2-10× improvement for partial dependencies
tags: async, parallelization, dependencies, better-all, optimization
---

## Use better-all for Dependency-Based Parallelization

**Impact: CRITICAL (2-10× improvement for partial dependencies)**

For operations with partial dependencies, use `better-all` to maximize parallelism. It automatically starts each task at the earliest possible moment, avoiding unnecessary sequential waterfalls while respecting dependencies.

**Incorrect (unnecessary sequential execution):**

```typescript
app.get('/dashboard/:userId', async ({ params }) => {
  // Step 1: Fetch user (100ms)
  const user = await db.users.findUnique({ where: { id: params.userId } })

  // Step 2: Fetch profile and posts in parallel (100ms)
  // But config could have started earlier!
  const [profile, config] = await Promise.all([
    db.profiles.findUnique({ where: { userId: user.id } }),
    db.configs.findUnique({ where: { key: 'dashboard' } })
  ])

  // Step 3: Fetch posts using user.id (100ms)
  const posts = await db.posts.findMany({
    where: { userId: user.id },
    take: config.postsLimit
  })

  return { user, profile, posts, config }
  // Total: 300ms (user → profile+config → posts)
})
```

**Correct (optimal parallelization with better-all):**

```typescript
import { all } from 'better-all'

app.get('/dashboard/:userId', async ({ params }) => {
  const { user, profile, config, posts } = await all({
    // Starts immediately
    async user() {
      return db.users.findUnique({ where: { id: params.userId } })
    },

    // Starts immediately (independent)
    async config() {
      return db.configs.findUnique({ where: { key: 'dashboard' } })
    },

    // Waits only for user.id
    async profile() {
      const u = await this.$.user
      return db.profiles.findUnique({ where: { userId: u.id } })
    },

    // Waits for both user.id and config.postsLimit
    async posts() {
      const [u, c] = await Promise.all([this.$.user, this.$.config])
      return db.posts.findMany({
        where: { userId: u.id },
        take: c.postsLimit
      })
    }
  })

  return { user, profile, posts, config }
  // Total: ~100ms (all overlap optimally)
  // user and config start together
  // profile and posts start as soon as their deps resolve
})
```

**Pattern: Authentication with parallel data fetching:**

```typescript
import { all } from 'better-all'

app.get('/api/report', async ({ headers }) => {
  const { user, permissions, settings, data } = await all({
    // Verify token
    async user() {
      const token = headers.authorization?.replace('Bearer ', '')
      return validateToken(token)
    },

    // Fetch permissions (depends on user)
    async permissions() {
      const u = await this.$.user
      if (!u) throw new Error('Unauthorized')
      return db.permissions.findMany({ where: { userId: u.id } })
    },

    // Fetch settings (depends on user)
    async settings() {
      const u = await this.$.user
      return db.settings.findUnique({ where: { userId: u.id } })
    },

    // Fetch data (depends on user + settings)
    async data() {
      const [u, s] = await Promise.all([this.$.user, this.$.settings])
      return fetchReportData(u.id, s.reportConfig)
    }
  })

  return { data, settings }
})
```

**Using in resolve lifecycle hook:**

```typescript
import { all } from 'better-all'

app
  .resolve(async ({ params }) => {
    return await all({
      // Fetch user
      async user() {
        return db.users.findUnique({ where: { id: params.userId } })
      },

      // Independent: fetch feature flags
      async features() {
        return getFeatureFlags()
      },

      // Depends on user: check subscription
      async subscription() {
        const u = await this.$.user
        if (!u) return null
        return db.subscriptions.findUnique({ where: { userId: u.id } })
      },

      // Depends on user + subscription: determine access level
      async access() {
        const [u, s] = await Promise.all([this.$.user, this.$.subscription])
        return calculateAccessLevel(u, s)
      }
    })
  })

  .get('/content/:userId', ({ user, access, features }) => {
    if (access.level < 2) throw new Error('Insufficient access')
    return getContent(user.id, features)
  }, {
    params: t.Object({
      userId: t.Numeric()
    })
  })
```

**Pattern: Multi-stage data pipeline:**

```typescript
import { all } from 'better-all'

app.post('/process', async ({ body }) => {
  const result = await all({
    // Validate input
    async validation() {
      return validateInput(body)
    },

    // Independent: check rate limits
    async rateLimit() {
      return checkRateLimit(body.userId)
    },

    // Independent: fetch configuration
    async config() {
      return getProcessingConfig()
    },

    // Depends on validation: sanitize data
    async sanitized() {
      await this.$.validation
      return sanitizeData(body)
    },

    // Depends on sanitized + config: process
    async processed() {
      const [data, cfg] = await Promise.all([this.$.sanitized, this.$.config])
      return processData(data, cfg)
    },

    // Depends on processed: save to database
    async saved() {
      const result = await this.$.processed
      return db.results.create({ data: result })
    }
  })

  return result.saved
}, {
  body: t.Object({
    userId: t.Number(),
    data: t.Any()
  })
})
```

**Error handling with allSettled:**

```typescript
import { allSettled } from 'better-all'

app.get('/aggregate/:userId', async ({ params }) => {
  const results = await allSettled({
    async user() {
      return db.users.findUnique({ where: { id: params.userId } })
    },

    async posts() {
      const u = await this.$.user
      return db.posts.findMany({ where: { userId: u.id } })
    },

    async comments() {
      const u = await this.$.user
      return db.comments.findMany({ where: { userId: u.id } })
    },

    async analytics() {
      const u = await this.$.user
      return fetchAnalytics(u.id)
    }
  })

  return {
    user: results.user.status === 'fulfilled' ? results.user.value : null,
    posts: results.posts.status === 'fulfilled' ? results.posts.value : [],
    comments: results.comments.status === 'fulfilled' ? results.comments.value : [],
    analytics: results.analytics.status === 'fulfilled' ? results.analytics.value : null
  }
})
```

**Complex dependency graph:**

```typescript
import { all } from 'better-all'

app.get('/complex/:orderId', async ({ params }) => {
  const data = await all({
    // Independent: start immediately
    async config() {
      return getConfig()
    },

    async order() {
      return db.orders.findUnique({ where: { id: params.orderId } })
    },

    // Depends on order
    async customer() {
      const o = await this.$.order
      return db.customers.findUnique({ where: { id: o.customerId } })
    },

    async items() {
      const o = await this.$.order
      return db.orderItems.findMany({ where: { orderId: o.id } })
    },

    // Depends on customer
    async address() {
      const c = await this.$.customer
      return db.addresses.findUnique({ where: { id: c.addressId } })
    },

    async loyalty() {
      const c = await this.$.customer
      return db.loyalty.findUnique({ where: { customerId: c.id } })
    },

    // Depends on items + config
    async pricing() {
      const [items, cfg] = await Promise.all([this.$.items, this.$.config])
      return calculatePricing(items, cfg.taxRate)
    },

    // Depends on customer + pricing
    async discount() {
      const [c, p] = await Promise.all([this.$.customer, this.$.pricing])
      return calculateDiscount(c.tier, p.total)
    }
  })

  return data
})
// Optimal parallelization: config/order start together,
// customer/items start when order resolves,
// address/loyalty start when customer resolves,
// pricing starts when items+config resolve,
// discount waits for customer+pricing
```

**Installation:**

```bash
bun add better-all
```

**Type Safety:**

TypeScript automatically infers the correct types for all dependencies accessed via `this.$`:

```typescript
const { user, posts } = await all({
  async user() {
    return { id: 1, name: 'John' } // Type: { id: number; name: string }
  },
  async posts() {
    const u = await this.$.user // u is correctly typed!
    return [`Post by ${u.name}`] // Type: string[]
  }
})
// user: { id: number; name: string }
// posts: string[]
```

Reference: [better-all on GitHub](https://github.com/shuding/better-all)
