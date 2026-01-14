---
title: Defer Reactive Reads to Usage Point
impact: MEDIUM
impactDescription: reduces tracking overhead
tags: reactivity, computed, defer
---

## Defer Reactive Reads to Usage Point

Read reactive values only when needed to avoid unnecessary dependency tracking.

**Incorrect (reads value even when not used):**

```vue
<script setup lang="ts">
const user = ref<User>()
const loading = ref(true)

// Reads user.value even when loading
const displayName = computed(() => {
  const u = user.value
  if (loading.value) return 'Loading...'
  return u?.name || 'Unknown'
})
</script>
```

**Correct (reads value only when needed):**

```vue
<script setup lang="ts">
const user = ref<User>()
const loading = ref(true)

// Only accesses user.value when not loading
const displayName = computed(() => {
  if (loading.value) return 'Loading...'
  return user.value?.name || 'Unknown'
})
</script>
```

**In composables:**

```typescript
// Incorrect
export function useUserProfile(userId: Ref<string>) {
  const profile = ref<Profile>()

  // Always tracks userId, even when disabled
  const fetchProfile = async (enabled: boolean) => {
    const id = userId.value
    if (!enabled) return
    profile.value = await fetch(`/api/users/${id}`)
  }

  return { profile, fetchProfile }
}

// Correct
export function useUserProfile(userId: Ref<string>) {
  const profile = ref<Profile>()

  // Only tracks userId when enabled
  const fetchProfile = async (enabled: boolean) => {
    if (!enabled) return
    const id = userId.value
    profile.value = await fetch(`/api/users/${id}`)
  }

  return { profile, fetchProfile }
}
```

**With conditional rendering:**

```vue
<script setup lang="ts">
const showDetails = ref(false)
const user = ref<User>()

// Only computes when showDetails is true
const userDetails = computed(() => {
  if (!showDetails.value) return null

  // Expensive computation only runs when needed
  return {
    ...user.value,
    avatar: computeAvatar(user.value),
    stats: computeStats(user.value)
  }
})
</script>
```

Reference: [Vue 3 Reactivity Fundamentals](https://vuejs.org/guide/essentials/reactivity-fundamentals.html)
