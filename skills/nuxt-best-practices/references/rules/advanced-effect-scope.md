---
title: Use effectScope for Grouped Effects
impact: LOW
impactDescription: Better lifecycle management for grouped effects
tags: advanced, effect-scope, lifecycle
---

## Use effectScope for Grouped Effects

**Impact: LOW**

Use `effectScope` to group and dispose of multiple effects together.

**Without effectScope:**

```vue
<script setup>
let watchStops: (() => void)[] = []

const startMonitoring = () => {
  watchStops.push(watch(data, () => { /* ... */ }))
  watchStops.push(watchEffect(() => { /* ... */ }))

  const interval = setInterval(() => { /* ... */ }, 1000)
  watchStops.push(() => clearInterval(interval))
}

const stopMonitoring = () => {
  watchStops.forEach(stop => stop())
  watchStops = []
}
</script>
```

**With effectScope:**

```vue
<script setup>
import { effectScope } from 'vue'

let scope: EffectScope | null = null

const startMonitoring = () => {
  scope = effectScope()

  scope.run(() => {
    watch(data, () => { /* ... */ })
    watchEffect(() => { /* ... */ })

    const interval = setInterval(() => { /* ... */ }, 1000)

    onScopeDispose(() => {
      clearInterval(interval)
    })
  })
}

const stopMonitoring = () => {
  scope?.stop()
  scope = null
}
</script>
```

All watchers and effects are automatically cleaned up when `scope.stop()` is called.

**Nested scopes:**

```typescript
const parentScope = effectScope()

parentScope.run(() => {
  const childScope = effectScope()

  childScope.run(() => {
    watch(data, () => { /* ... */ })
  })

  // Stop only child scope
  childScope.stop()
})

// Stop parent scope (and all children)
parentScope.stop()
```

**Use cases:**
- Plugin systems
- Dynamic feature toggles
- Complex lifecycle management
- Reusable composables with cleanup

Reference: [Vue effectScope](https://vuejs.org/api/reactivity-advanced.html#effectscope)
