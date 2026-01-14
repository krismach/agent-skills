---
title: Use Computed for Derived State
impact: MEDIUM
impactDescription: caches derived values
tags: reactivity, computed, optimization
---

## Use Computed for Derived State

Use `computed()` for expensive derived values to cache results and avoid unnecessary recalculations.

**Incorrect (recalculates on every access):**

```vue
<script setup lang="ts">
const items = ref<Item[]>([])
const query = ref('')

// Recalculates on every template access
const filteredItems = () => {
  return items.value.filter(item =>
    item.name.toLowerCase().includes(query.value.toLowerCase())
  )
}
</script>

<template>
  <!-- Called multiple times per render -->
  <div v-for="item in filteredItems()" :key="item.id">
    {{ item.name }}
  </div>
  <p>Found {{ filteredItems().length }} items</p>
</template>
```

**Correct (caches and only recomputes when dependencies change):**

```vue
<script setup lang="ts">
const items = ref<Item[]>([])
const query = ref('')

// Caches result, only recomputes when items or query changes
const filteredItems = computed(() => {
  return items.value.filter(item =>
    item.name.toLowerCase().includes(query.value.toLowerCase())
  )
})
</script>

<template>
  <div v-for="item in filteredItems" :key="item.id">
    {{ item.name }}
  </div>
  <p>Found {{ filteredItems.length }} items</p>
</template>
```

**For expensive transformations:**

```vue
<script setup lang="ts">
const users = ref<User[]>([])

// Builds index only when users change
const userIndex = computed(() => {
  const index = new Map()
  users.value.forEach(user => {
    index.set(user.id, user)
  })
  return index
})

// Uses cached index
const getUser = (id: string) => userIndex.value.get(id)
</script>
```

Reference: [Vue 3 Computed Properties](https://vuejs.org/guide/essentials/computed.html)
