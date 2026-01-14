---
title: Lazy Load Heavy Components
impact: CRITICAL
impactDescription: Reduces initial bundle by 20-40%
tags: bundle, lazy, code-splitting
---

## Lazy Load Heavy Components

**Impact: CRITICAL**

Use `defineAsyncComponent` or lazy component syntax for large components not needed on initial render.

**Incorrect: Monaco bundles with main chunk ~300KB**

```vue
<script setup>
import MonacoEditor from '~/components/MonacoEditor.vue'
</script>

<template>
  <MonacoEditor v-if="showEditor" />
</template>
```

**Correct: Monaco loads on demand**

```vue
<script setup>
const MonacoEditor = defineAsyncComponent(() =>
  import('~/components/MonacoEditor.vue')
)
</script>

<template>
  <MonacoEditor v-if="showEditor" />
</template>
```

**Alternative: Lazy prefix (auto-imports)**

```vue
<template>
  <LazyMonacoEditor v-if="showEditor" />
</template>
```

Any component prefixed with `Lazy` is automatically code-split by Nuxt.

Reference: [Nuxt Lazy Components](https://nuxt.com/docs/guide/directory-structure/components#lazy-components)
