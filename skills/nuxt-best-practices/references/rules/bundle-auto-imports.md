---
title: Leverage Auto-Imports
impact: CRITICAL
impactDescription: Enables better tree-shaking and reduces bundle size
tags: bundle, auto-imports, tree-shaking
---

## Leverage Auto-Imports

**Impact: CRITICAL**

Nuxt 3 auto-imports components, composables, and utilities. Use this instead of manual imports to enable better tree-shaking.

**Incorrect: manual imports**

```vue
<script setup>
import { ref, computed, watch } from 'vue'
import { useFetch } from '#app'
import MyComponent from '~/components/MyComponent.vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>
```

**Correct: auto-imports**

```vue
<script setup>
const count = ref(0)
const doubled = computed(() => count.value * 2)
const { data } = useFetch('/api/data')
</script>

<template>
  <MyComponent />
</template>
```

Auto-imports enable better tree-shaking and reduce bundle size by eliminating unused imports.

**Configure auto-imports:**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  imports: {
    dirs: [
      'composables/**',
      'utils/**'
    ]
  }
})
```

Reference: [Nuxt Auto-imports](https://nuxt.com/docs/guide/concepts/auto-imports)
