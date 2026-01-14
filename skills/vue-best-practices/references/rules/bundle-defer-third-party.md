---
title: Defer Non-Critical Third-Party Scripts
impact: HIGH
impactDescription: improves TTI and main thread availability
tags: bundle, third-party, analytics, defer
---

## Defer Non-Critical Third-Party Scripts

Load analytics, monitoring, and chat widgets after the main application is interactive.

**Incorrect (blocks main thread):**

```vue
<script setup lang="ts">
import Analytics from '@segment/analytics-next'
import * as Sentry from '@sentry/vue'

// Blocks initial load
const analytics = Analytics.load({ writeKey: 'key' })
Sentry.init({ dsn: 'dsn' })
</script>
```

**Correct (deferred loading):**

```vue
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  // Load after interactive
  if (import.meta.client) {
    void import('@segment/analytics-next').then(({ default: Analytics }) => {
      Analytics.load({ writeKey: 'key' })
    })

    void import('@sentry/vue').then(Sentry => {
      Sentry.init({ dsn: 'dsn' })
    })
  }
})
</script>
```

**In Nuxt 3 with plugins:**

```typescript
// plugins/analytics.client.ts
export default defineNuxtPlugin({
  name: 'analytics',
  parallel: true,
  async setup() {
    // Defer until after hydration
    await new Promise(resolve => setTimeout(resolve, 2000))

    const Analytics = await import('@segment/analytics-next')
    const analytics = Analytics.default.load({ writeKey: 'key' })

    return {
      provide: {
        analytics
      }
    }
  }
})
```

**Use useHead for external scripts:**

```vue
<script setup lang="ts">
useHead({
  script: [
    {
      src: 'https://cdn.example.com/widget.js',
      defer: true, // or async: true
      tagPosition: 'bodyClose'
    }
  ]
})
</script>
```

This ensures third-party code doesn't block your application's critical rendering path.

Reference: [Nuxt 3 Plugins](https://nuxt.com/docs/guide/directory-structure/plugins)
