---
title: Use Stable Keys for List Rendering
impact: MEDIUM
impactDescription: prevents unnecessary re-renders
tags: rendering, v-for, keys, lists
---

## Use Stable Keys for List Rendering

Always use stable, unique keys for `v-for` to help Vue track element identity.

**Incorrect (uses index as key):**

```vue
<script setup lang="ts">
const items = ref([
  { name: 'Alice' },
  { name: 'Bob' },
  { name: 'Charlie' }
])

const addItem = () => {
  items.value.unshift({ name: 'New' })
}
</script>

<template>
  <!-- Keys shift when new item added, causing re-renders -->
  <div v-for="(item, index) in items" :key="index">
    <ExpensiveComponent :name="item.name" />
  </div>
</template>
```

When a new item is added at the beginning, all indices shift, causing Vue to re-render all components.

**Correct (uses stable unique key):**

```vue
<script setup lang="ts">
const items = ref([
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
  { id: 3, name: 'Charlie' }
])

let nextId = 4
const addItem = () => {
  items.value.unshift({ id: nextId++, name: 'New' })
}
</script>

<template>
  <!-- Keys remain stable, only new item is rendered -->
  <div v-for="item in items" :key="item.id">
    <ExpensiveComponent :name="item.name" />
  </div>
</template>
```

**For dynamic forms:**

```vue
<script setup lang="ts">
import { nanoid } from 'nanoid'

const fields = ref([
  { id: nanoid(), value: '' }
])

const addField = () => {
  fields.value.push({ id: nanoid(), value: '' })
}
</script>

<template>
  <div v-for="field in fields" :key="field.id">
    <input v-model="field.value" />
  </div>
</template>
```

**When index is acceptable:**
- List never reorders
- List is static (no adds/removes)
- Items have no internal state
- Performance is not critical

Reference: [Vue 3 List Rendering](https://vuejs.org/guide/essentials/list.html#maintaining-state-with-key)
