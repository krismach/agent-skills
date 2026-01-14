---
title: Use Nitro Route Rules
impact: HIGH
impactDescription: Optimizes caching and rendering strategies
tags: server, nitro, route-rules, caching
---

## Use Nitro Route Rules

**Impact: HIGH**

Configure route behavior using route rules for static generation, caching, and redirects.

**Configuration:**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // Static generation
    '/about': { prerender: true },

    // SWR caching (stale-while-revalidate)
    '/api/posts': { swr: 60 }, // Cache for 60 seconds

    // ISR (Incremental Static Regeneration)
    '/blog/**': { isr: 3600 }, // Regenerate every hour

    // Client-side only
    '/dashboard/**': { ssr: false },

    // Redirect
    '/old-path': { redirect: '/new-path' },

    // Cache headers
    '/api/data': {
      cache: {
        maxAge: 60 * 60, // 1 hour
        staleMaxAge: 60 * 60 * 24 // 24 hours stale
      }
    }
  }
})
```

**Per-page route rules:**

```vue
<!-- pages/blog/[slug].vue -->
<script setup>
defineRouteRules({
  isr: 3600, // Regenerate every hour
  cache: {
    maxAge: 60 * 60
  }
})
</script>
```

Route rules allow fine-grained control over caching, rendering, and delivery strategies.

Reference: [Nuxt Route Rules](https://nuxt.com/docs/guide/going-further/experimental-features#inlinerouterules)
