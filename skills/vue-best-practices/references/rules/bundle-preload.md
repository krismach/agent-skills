---
title: Preload Based on User Intent
impact: CRITICAL
impactDescription: reduces perceived latency
tags: bundle, preload, user-intent, hover
---

## Preload Based on User Intent

Preload heavy bundles before they're needed to reduce perceived latency.

**Example (preload on hover/focus):**

```vue
<script setup lang="ts">
const preloadEditor = () => {
  if (import.meta.client) {
    void import('./monaco-editor')
  }
}

const openEditor = () => {
  // Editor already preloaded
  showEditor.value = true
}
</script>

<template>
  <button
    @mouseenter="preloadEditor"
    @focus="preloadEditor"
    @click="openEditor"
  >
    Open Editor
  </button>
</template>
```

**Example (preload when feature flag is enabled):**

```vue
<script setup lang="ts">
const flags = useFeatureFlags()

watchEffect(() => {
  if (flags.value.editorEnabled && import.meta.client) {
    void import('./monaco-editor').then(mod => mod.init())
  }
})
</script>
```

**In Nuxt 3:**

```vue
<script setup lang="ts">
const flags = useFeatureFlags()

watchEffect(() => {
  if (flags.value.editorEnabled && process.client) {
    void import('./monaco-editor').then(mod => mod.init())
  }
})
</script>
```

The `import.meta.client` or `process.client` check prevents bundling preloaded modules for SSR, optimizing server bundle size and build speed.

Reference: [Vue 3 Dynamic Imports](https://vuejs.org/guide/components/async.html#using-with-suspense)
