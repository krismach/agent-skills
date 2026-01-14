---
title: Use Shallow Refs for Large Objects
impact: MEDIUM
impactDescription: Reduces reactivity overhead for large objects
tags: reactivity, shallow-ref, performance
---

## Use Shallow Refs for Large Objects

**Impact: MEDIUM**

For large objects where you only mutate top-level properties, use `shallowRef` to skip deep reactivity.

**Incorrect: deep reactivity for large object**

```vue
<script setup>
const largeConfig = ref({
  settings: { /* 1000s of nested properties */ },
  theme: { /* deep nested object */ }
})

// Triggers deep reactivity tracking
largeConfig.value.settings.foo = 'bar'
</script>
```

**Correct: shallow reactivity**

```vue
<script setup>
const largeConfig = shallowRef({
  settings: { /* 1000s of nested properties */ },
  theme: { /* deep nested object */ }
})

// Replace entire object to trigger update
largeConfig.value = {
  ...largeConfig.value,
  settings: { ...largeConfig.value.settings, foo: 'bar' }
}
</script>
```

**Use shallowRef when:**
- Object is very large
- You only update top-level properties
- Deep reactivity isn't needed

**Manual trigger with triggerRef:**

```vue
<script setup>
const state = shallowRef({ count: 0 })

function increment() {
  // Mutate in place
  state.value.count++
  // Manually trigger update
  triggerRef(state)
}
</script>
```

Reference: [Vue shallowRef](https://vuejs.org/api/reactivity-advanced.html#shallowref)
