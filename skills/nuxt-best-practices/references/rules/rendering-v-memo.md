---
title: Use v-memo for Expensive Lists
impact: MEDIUM
impactDescription: Skips re-rendering when dependencies haven't changed
tags: rendering, v-memo, lists
---

## Use v-memo for Expensive Lists

**Impact: MEDIUM**

Use `v-memo` to skip re-rendering list items when dependencies haven't changed.

**Without v-memo:**

```vue
<template>
  <div v-for="item in items" :key="item.id">
    <ExpensiveComponent :data="item" />
  </div>
</template>
```

Every item re-renders when `items` array changes, even if individual items haven't changed.

**With v-memo:**

```vue
<template>
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id, item.updatedAt]"
  >
    <ExpensiveComponent :data="item" />
  </div>
</template>
```

Item only re-renders when `id` or `updatedAt` changes.

**Example with selection:**

```vue
<script setup>
const items = ref([/* items */])
const selectedId = ref(null)
</script>

<template>
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id === selectedId, item.name]"
    @click="selectedId = item.id"
  >
    <div :class="{ selected: item.id === selectedId }">
      {{ item.name }}
    </div>
  </div>
</template>
```

**When to use v-memo:**
- Long lists with expensive child components
- Virtual scrolling alternatives
- Lists where most items don't change between updates

**When NOT to use v-memo:**
- Simple lists (overhead not worth it)
- Every item changes on update
- Already using virtual scrolling

Reference: [Vue v-memo](https://vuejs.org/api/built-in-directives.html#v-memo)
