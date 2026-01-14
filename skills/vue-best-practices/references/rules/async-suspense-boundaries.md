---
title: Strategic Suspense Boundaries
impact: HIGH
impactDescription: faster initial paint
tags: async, suspense, streaming, layout-shift
---

## Strategic Suspense Boundaries

Instead of awaiting data in async components before returning template, use Suspense boundaries to show the wrapper UI faster while data loads.

**Incorrect (wrapper blocked by data fetching):**

```vue
<script setup lang="ts">
const data = await fetchData() // Blocks entire page
</script>

<template>
  <div>
    <div>Sidebar</div>
    <div>Header</div>
    <div>
      <DataDisplay :data="data" />
    </div>
    <div>Footer</div>
  </div>
</template>
```

The entire layout waits for data even though only the middle section needs it.

**Correct (wrapper shows immediately, data streams in):**

```vue
<!-- Page.vue -->
<template>
  <div>
    <div>Sidebar</div>
    <div>Header</div>
    <div>
      <Suspense>
        <template #default>
          <DataDisplay />
        </template>
        <template #fallback>
          <Skeleton />
        </template>
      </Suspense>
    </div>
    <div>Footer</div>
  </div>
</template>

<!-- DataDisplay.vue -->
<script setup lang="ts">
const data = await fetchData() // Only blocks this component
</script>

<template>
  <div>{{ data.content }}</div>
</template>
```

Sidebar, Header, and Footer render immediately. Only DataDisplay waits for data.

**When NOT to use this pattern:**

- Critical data needed for layout decisions (affects positioning)
- SEO-critical content above the fold
- Small, fast queries where suspense overhead isn't worth it
- When you want to avoid layout shift (loading → content jump)

**Trade-off:** Faster initial paint vs potential layout shift. Choose based on your UX priorities.

Reference: [Vue 3 Suspense](https://vuejs.org/guide/built-ins/suspense.html)
