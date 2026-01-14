---
title: Use Computed Instead of Methods
impact: MEDIUM
impactDescription: Caches results and avoids redundant computation
tags: reactivity, computed, vue
---

## Use Computed Instead of Methods

**Impact: MEDIUM**

Computed properties cache results; methods recompute every time.

**Incorrect: recalculates on every render**

```vue
<script setup>
const items = ref([/* large array */])

function filteredItems() {
  return items.value.filter(item => item.active)
}
</script>

<template>
  <div v-for="item in filteredItems()" :key="item.id">
    {{ item.name }}
  </div>
</template>
```

`filteredItems()` runs on every render, even if `items` didn't change.

**Correct: cached and reactive**

```vue
<script setup>
const items = ref([/* large array */])

const filteredItems = computed(() => {
  return items.value.filter(item => item.active)
})
</script>

<template>
  <div v-for="item in filteredItems" :key="item.id">
    {{ item.name }}
  </div>
</template>
```

Computed runs only when `items` changes.

**When to use methods:**
- Side effects (API calls, logging)
- Event handlers
- Actions that should run every time

**When to use computed:**
- Derived state
- Transformations
- Filtered/sorted data

Reference: [Vue Computed Properties](https://vuejs.org/guide/essentials/computed.html)
