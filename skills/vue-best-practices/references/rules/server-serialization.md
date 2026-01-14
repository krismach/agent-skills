---
title: Minimize Serialization at SSR Boundaries
impact: HIGH
impactDescription: reduces data transfer size
tags: server, ssr, serialization, payload
---

## Minimize Serialization at SSR Boundaries

The server/client boundary serializes all data passed to the client. Only pass fields that the client actually uses.

**Incorrect (serializes all 50 fields):**

```vue
<script setup lang="ts">
const { data: user } = await useFetch('/api/user')  // 50 fields
</script>

<template>
  <Profile :user="user" />  <!-- passes entire object -->
</template>

<!-- components/Profile.vue -->
<script setup lang="ts">
defineProps<{ user: User }>()
</script>

<template>
  <div>{{ user.name }}</div>  <!-- uses 1 field -->
</template>
```

**Correct (serializes only 1 field):**

```vue
<script setup lang="ts">
const { data: user } = await useFetch('/api/user', {
  transform: (data) => ({ name: data.name })  // Pick only needed fields
})
</script>

<template>
  <Profile :name="user.name" />
</template>

<!-- components/Profile.vue -->
<script setup lang="ts">
defineProps<{ name: string }>()
</script>

<template>
  <div>{{ name }}</div>
</template>
```

**Alternative with API transformation:**

```typescript
// server/api/user/profile.get.ts
export default defineEventHandler(async (event) => {
  const user = await db.user.findUnique({
    where: { id: event.context.userId },
    select: {
      // Only select fields needed by client
      name: true,
      email: true,
      avatar: true
    }
  })

  return user
})
```

**With computed properties:**

```vue
<script setup lang="ts">
const { data: user } = await useFetch('/api/user')

// Extract only needed data
const profileData = computed(() => ({
  name: user.value?.name,
  avatar: user.value?.avatar
}))
</script>

<template>
  <Profile v-bind="profileData" />
</template>
```

This reduces the payload size transferred from server to client, improving initial load time and reducing bandwidth usage.

Reference: [Nuxt 3 useFetch](https://nuxt.com/docs/api/composables/use-fetch)
