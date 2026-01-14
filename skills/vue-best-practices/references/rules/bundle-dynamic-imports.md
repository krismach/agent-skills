---
title: Dynamic Imports for Heavy Components
impact: CRITICAL
impactDescription: directly affects TTI and LCP
tags: bundle, dynamic-import, code-splitting, async-component
---

## Dynamic Imports for Heavy Components

Use `defineAsyncComponent()` to lazy-load large components not needed on initial render.

**Incorrect (Monaco bundles with main chunk ~300KB):**

```vue
<script setup lang="ts">
import { MonacoEditor } from './monaco-editor'
</script>

<template>
  <MonacoEditor :value="code" />
</template>
```

**Correct (Monaco loads on demand):**

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const MonacoEditor = defineAsyncComponent(() =>
  import('./monaco-editor').then(m => m.MonacoEditor)
)
</script>

<template>
  <Suspense>
    <template #default>
      <MonacoEditor :value="code" />
    </template>
    <template #fallback>
      <div>Loading editor...</div>
    </template>
  </Suspense>
</template>
```

**Alternative with error handling:**

```typescript
const MonacoEditor = defineAsyncComponent({
  loader: () => import('./monaco-editor').then(m => m.MonacoEditor),
  loadingComponent: EditorSkeleton,
  errorComponent: EditorError,
  delay: 200, // delay before showing loading component
  timeout: 10000 // timeout after 10s
})
```

**In Nuxt 3, use the Lazy prefix:**

```vue
<template>
  <LazyMonacoEditor :value="code" />
</template>
```

Nuxt automatically code-splits components with the `Lazy` prefix without any additional configuration.

Reference: [Vue 3 Async Components](https://vuejs.org/guide/components/async.html)
