---
title: Parallel Data Fetching with Composables
impact: HIGH
impactDescription: eliminates server-side waterfalls
tags: server, ssr, parallel-fetching, composables
---

## Parallel Data Fetching with Composables

Nuxt executes data fetching composables in the order they appear. Restructure to parallelize independent data fetching.

**Incorrect (Sidebar waits for header fetch to complete):**

```vue
<script setup lang="ts">
const { data: header } = await useFetch('/api/header')
const { data: items } = await useFetch('/api/sidebar')
</script>

<template>
  <div>
    <div>{{ header }}</div>
    <nav>{{ items }}</nav>
  </div>
</template>
```

**Correct (both fetch simultaneously):**

```vue
<script setup lang="ts">
// Start both fetches immediately without awaiting
const headerPromise = useFetch('/api/header')
const itemsPromise = useFetch('/api/sidebar')

// Await them together
const [{ data: header }, { data: items }] = await Promise.all([
  headerPromise,
  itemsPromise
])
</script>

<template>
  <div>
    <div>{{ header }}</div>
    <nav>{{ items }}</nav>
  </div>
</template>
```

**Alternative with async components:**

```vue
<!-- pages/index.vue -->
<template>
  <div>
    <Header />
    <Sidebar />
  </div>
</template>

<!-- components/Header.vue -->
<script setup lang="ts">
const { data: header } = await useFetch('/api/header')
</script>

<template>
  <div>{{ header }}</div>
</template>

<!-- components/Sidebar.vue -->
<script setup lang="ts">
const { data: items } = await useFetch('/api/sidebar')
</script>

<template>
  <nav>{{ items }}</nav>
</template>
```

Nuxt automatically parallelizes data fetching for sibling components.

**With lazy loading:**

```vue
<template>
  <div>
    <LazyHeader />
    <LazySidebar />
  </div>
</template>
```

Reference: [Nuxt 3 Data Fetching Composables](https://nuxt.com/docs/getting-started/data-fetching)
