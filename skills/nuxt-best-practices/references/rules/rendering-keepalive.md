---
title: Use KeepAlive for Component Caching
impact: MEDIUM
impactDescription: Preserves component state and avoids re-rendering
tags: rendering, keepalive, caching
---

## Use KeepAlive for Component Caching

**Impact: MEDIUM**

Use `<KeepAlive>` to preserve component state and avoid re-rendering expensive components.

**Without KeepAlive:**

```vue
<template>
  <component :is="currentTab" />
</template>
```

Component state is lost and component re-renders when switching tabs.

**With KeepAlive:**

```vue
<template>
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

Component state persists when switching tabs.

**With max cache:**

```vue
<template>
  <KeepAlive :max="10">
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

Limits the number of cached component instances to 10.

**With include/exclude:**

```vue
<template>
  <KeepAlive :include="['TabA', 'TabB']" :exclude="['TabC']">
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

**Lifecycle hooks:**

```vue
<script setup>
onActivated(() => {
  // Called when component is activated from cache
  console.log('Component activated')
})

onDeactivated(() => {
  // Called when component is deactivated and cached
  console.log('Component deactivated')
})
</script>
```

**Use KeepAlive for:**
- Tab systems
- Multi-step forms
- Expensive data visualizations
- Editor components

Reference: [Vue KeepAlive](https://vuejs.org/guide/built-ins/keep-alive.html)
