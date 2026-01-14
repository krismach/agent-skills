---
title: Use useFetch for API Calls
impact: MEDIUM-HIGH
impactDescription: Enables SSR, reactivity, and deduplication
tags: client, data-fetching, useFetch
---

## Use useFetch for API Calls

**Impact: MEDIUM-HIGH**

Always use `useFetch` or `useAsyncData` instead of manual `$fetch` for better SSR support and automatic reactivity.

**Incorrect: no SSR, no reactivity**

```vue
<script setup>
const data = ref(null)

onMounted(async () => {
  data.value = await $fetch('/api/data')
})
</script>
```

**Correct: SSR + reactivity**

```vue
<script setup>
const { data } = useFetch('/api/data')
</script>
```

**Benefits:**
- Executes on server during SSR
- Automatically reactive
- Deduplicates requests
- Handles loading and error states

**With options:**

```vue
<script setup>
const { data, pending, error, refresh } = useFetch('/api/data', {
  key: 'my-data',
  lazy: true,
  server: true,
  transform: (data) => data.items,
  watch: [searchQuery]
})
</script>
```

**For computed URLs:**

```vue
<script setup>
const userId = ref('123')
const { data } = useFetch(() => `/api/users/${userId.value}`)
// Automatically refetches when userId changes
</script>
```

Reference: [Nuxt useFetch](https://nuxt.com/docs/api/composables/use-fetch)
