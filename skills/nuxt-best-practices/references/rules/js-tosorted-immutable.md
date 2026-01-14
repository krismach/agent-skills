---
title: Use toSorted() Instead of sort() for Immutability
impact: LOW-MEDIUM
impactDescription: Prevents bugs from array mutation
tags: javascript, immutability, arrays
---

## Use toSorted() Instead of sort() for Immutability

**Impact: LOW-MEDIUM**

`.sort()` mutates the array in place, which can cause bugs with Vue reactivity. Use `.toSorted()` to create a new sorted array without mutation.

**Incorrect: mutates original array**

```vue
<script setup>
const users = ref([/* users */])

const sorted = computed(() =>
  users.value.sort((a, b) => a.name.localeCompare(b.name))
)
</script>
```

This mutates the `users` array, which can cause unexpected behavior and bugs.

**Correct: creates new array**

```vue
<script setup>
const users = ref([/* users */])

const sorted = computed(() =>
  users.value.toSorted((a, b) => a.name.localeCompare(b.name))
)
</script>
```

**Why this matters in Vue:**

```vue
<script setup>
const items = ref([3, 1, 2])

// Incorrect: mutates reactive array
const sortItems = () => {
  items.value.sort()
  // This may not trigger reactivity properly
}

// Correct: creates new array
const sortItems = () => {
  items.value = items.value.toSorted()
  // Properly triggers reactivity
}
</script>
```

**Fallback for older browsers:**

```typescript
// Use spread operator as fallback
const sorted = [...items].sort((a, b) => a.value - b.value)
```

**Other immutable array methods:**

```typescript
// toSorted() - immutable sort
const sorted = arr.toSorted()

// toReversed() - immutable reverse
const reversed = arr.toReversed()

// toSpliced() - immutable splice
const modified = arr.toSpliced(2, 1, 'new')

// with() - immutable index update
const updated = arr.with(0, 'new')
```

Reference: [MDN toSorted()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted)
