---
title: Use useFetch for Automatic Deduplication
impact: MEDIUM-HIGH
impactDescription: automatic deduplication
tags: client, usefetch, deduplication, data-fetching
---

## Use useFetch for Automatic Deduplication

`useFetch` in Nuxt enables request deduplication, caching, and revalidation across component instances.

**Incorrect (no deduplication, each instance fetches):**

```vue
<script setup lang="ts">
const users = ref([])

onMounted(async () => {
  const response = await fetch('/api/users')
  users.value = await response.json()
})
</script>
```

**Correct (multiple instances share one request):**

```vue
<script setup lang="ts">
const { data: users } = await useFetch('/api/users', {
  key: 'users-list'
})
</script>
```

**For client-only fetching:**

```vue
<script setup lang="ts">
const { data: users } = useFetch('/api/users', {
  key: 'users-list',
  server: false // Only fetch on client
})
</script>
```

**For mutations:**

```vue
<script setup lang="ts">
const updateUser = async () => {
  await $fetch('/api/user', {
    method: 'POST',
    body: userData
  })

  // Refresh the cached data
  await refreshNuxtData('users-list')
}
</script>
```

**With VueUse composables:**

```typescript
import { useFetch as vueUseFetch } from '@vueuse/core'

const { data, isFetching } = vueUseFetch('/api/users', {
  refetch: true
}).json()
```

**Lazy loading with useLazyFetch:**

```vue
<script setup lang="ts">
const { data: users, pending, refresh } = useLazyFetch('/api/users', {
  key: 'users-list'
})

// Trigger fetch manually
const loadUsers = () => refresh()
</script>
```

Reference: [Nuxt 3 useFetch](https://nuxt.com/docs/api/composables/use-fetch)
