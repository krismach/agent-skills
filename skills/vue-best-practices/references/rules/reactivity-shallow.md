---
title: Use shallowRef for Large Data Structures
impact: MEDIUM
impactDescription: reduces reactive overhead
tags: reactivity, shallow, performance
---

## Use shallowRef for Large Data Structures

Use `shallowRef` or `shallowReactive` for large data structures where you only need top-level reactivity.

**Incorrect (deep reactivity for large arrays):**

```vue
<script setup lang="ts">
// Makes every nested property reactive
const data = ref({
  items: Array(10000).fill(0).map((_, i) => ({
    id: i,
    name: `Item ${i}`,
    metadata: { count: 0, timestamp: Date.now() }
  }))
})

// Triggers expensive reactive tracking for entire array
const updateItem = (index: number) => {
  data.value.items[index].metadata.count++
}
</script>
```

**Correct (shallow reactivity):**

```vue
<script setup lang="ts">
// Only the top-level reference is reactive
const data = shallowRef({
  items: Array(10000).fill(0).map((_, i) => ({
    id: i,
    name: `Item ${i}`,
    metadata: { count: 0, timestamp: Date.now() }
  }))
})

// Replace entire reference to trigger update
const updateItem = (index: number) => {
  const newData = { ...data.value }
  newData.items[index].metadata.count++
  data.value = newData
}
</script>
```

**With shallowReactive for objects:**

```vue
<script setup lang="ts">
// Only first-level properties are reactive
const state = shallowReactive({
  largeArray: Array(10000).fill(0),
  config: { /* large config object */ },
  cache: new Map()
})

// Trigger update by replacing property
const updateArray = (newArray: number[]) => {
  state.largeArray = newArray
}
</script>
```

**When to use:**
- Large arrays or maps (>1000 items)
- Data from external APIs that doesn't need nested reactivity
- Performance-critical lists
- Integration with non-reactive libraries

Reference: [Vue 3 Reactivity API](https://vuejs.org/api/reactivity-advanced.html#shallowref)
