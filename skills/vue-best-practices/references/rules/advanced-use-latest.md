---
title: useLatestValue for Stable Callback Refs
impact: LOW
impactDescription: prevents watcher re-runs
tags: advanced, composables, refs, optimization
---

## useLatestValue for Stable Callback Refs

Access latest values in callbacks without adding them to watch dependencies. Prevents watcher re-runs while avoiding stale closures.

**Implementation:**

```typescript
// composables/useLatestValue.ts
export function useLatestValue<T>(value: T | Ref<T>) {
  const valueRef = ref(unref(value)) as Ref<T>

  watch(() => unref(value), (newValue) => {
    valueRef.value = newValue
  })

  return valueRef
}
```

**Incorrect (watcher re-runs on every callback change):**

```vue
<script setup lang="ts">
const props = defineProps<{
  onSearch: (query: string) => void
}>()

const query = ref('')

// Re-runs whenever onSearch prop changes
watch(query, (newQuery) => {
  props.onSearch(newQuery)
}, { immediate: false })
</script>
```

**Correct (stable watcher, fresh callback):**

```vue
<script setup lang="ts">
const props = defineProps<{
  onSearch: (query: string) => void
}>()

const query = ref('')
const onSearchRef = useLatestValue(() => props.onSearch)

// Only re-runs when query changes
watch(query, (newQuery) => {
  onSearchRef.value(newQuery)
})
</script>
```

**With debounce:**

```vue
<script setup lang="ts">
const props = defineProps<{
  onSearch: (query: string) => void
}>()

const query = ref('')
const onSearchRef = useLatestValue(() => props.onSearch)

// Stable debounce, always calls latest callback
const debouncedSearch = useDebounceFn(() => {
  onSearchRef.value(query.value)
}, 300)

watch(query, debouncedSearch)
</script>
```

**Alternative with watchEffect and toRefs:**

```vue
<script setup lang="ts">
const props = defineProps<{
  onSearch: (query: string) => void
}>()

const { onSearch } = toRefs(props)
const query = ref('')

// watchEffect tracks dependencies automatically
watchEffect(() => {
  if (query.value) {
    onSearch.value(query.value)
  }
})
</script>
```

**Use VueUse's useLatest:**

```typescript
import { useLatest } from '@vueuse/core'

const props = defineProps<{
  onSearch: (query: string) => void
}>()

const query = ref('')
const onSearchRef = useLatest(props.onSearch)

watch(query, (newQuery) => {
  onSearchRef.value(newQuery)
})
```

Reference: [VueUse useLatest](https://vueuse.org/shared/useLatest/)
