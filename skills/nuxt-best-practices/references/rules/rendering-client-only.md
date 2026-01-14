---
title: Use ClientOnly for Browser-Specific Code
impact: MEDIUM
impactDescription: Prevents hydration mismatches
tags: rendering, hydration, client-only
---

## Use ClientOnly for Browser-Specific Code

**Impact: MEDIUM**

Wrap browser-specific code with `<ClientOnly>` to prevent hydration mismatches.

**Incorrect: breaks SSR**

```vue
<script setup>
const theme = localStorage.getItem('theme') || 'light'
</script>

<template>
  <div :class="theme">Content</div>
</template>
```

Server-side rendering will fail because `localStorage` is undefined.

**Correct: no hydration mismatch**

```vue
<script setup>
const theme = ref('light')

onMounted(() => {
  theme.value = localStorage.getItem('theme') || 'light'
})
</script>

<template>
  <ClientOnly>
    <div :class="theme">Content</div>
  </ClientOnly>
</template>
```

**Alternative: prevent flicker with placeholder**

```vue
<template>
  <ClientOnly>
    <div :class="theme">Content</div>
    <template #fallback>
      <div class="light">Content</div>
    </template>
  </ClientOnly>
</template>
```

**For entire components:**

```vue
<template>
  <ClientOnly>
    <BrowserOnlyComponent />
  </ClientOnly>
</template>
```

Reference: [Nuxt ClientOnly](https://nuxt.com/docs/api/components/client-only)
