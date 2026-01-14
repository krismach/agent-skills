---
title: Prevent Hydration Mismatch Without Flickering
impact: MEDIUM
impactDescription: avoids visual flicker and hydration errors
tags: rendering, ssr, hydration, localStorage, flicker
---

## Prevent Hydration Mismatch Without Flickering

When rendering content that depends on client-side storage (localStorage, cookies), avoid both SSR breakage and post-hydration flickering by using ClientOnly component or synchronous scripts.

**Incorrect (breaks SSR):**

```vue
<script setup lang="ts">
// localStorage is not available on server - throws error
const theme = localStorage.getItem('theme') || 'light'
</script>

<template>
  <div :class="theme">
    <slot />
  </div>
</template>
```

Server-side rendering will fail because `localStorage` is undefined.

**Incorrect (visual flickering):**

```vue
<script setup lang="ts">
const theme = ref('light')

onMounted(() => {
  // Runs after hydration - causes visible flash
  const stored = localStorage.getItem('theme')
  if (stored) {
    theme.value = stored
  }
})
</script>

<template>
  <div :class="theme">
    <slot />
  </div>
</template>
```

Component first renders with default value (`light`), then updates after hydration, causing a visible flash.

**Correct (using ClientOnly in Nuxt):**

```vue
<script setup lang="ts">
const theme = ref('light')

if (process.client) {
  theme.value = localStorage.getItem('theme') || 'light'
}
</script>

<template>
  <ClientOnly>
    <div :class="theme">
      <slot />
    </div>
    <template #fallback>
      <div class="light">
        <slot />
      </div>
    </template>
  </ClientOnly>
</template>
```

**Correct (using synchronous script):**

```vue
<template>
  <div>
    <div id="theme-wrapper">
      <slot />
    </div>
    <script>
      (function() {
        try {
          var theme = localStorage.getItem('theme') || 'light';
          var el = document.getElementById('theme-wrapper');
          if (el) el.className = theme;
        } catch (e) {}
      })();
    </script>
  </div>
</template>
```

**In Nuxt with useHead:**

```vue
<script setup lang="ts">
useHead({
  script: [
    {
      innerHTML: `
        (function() {
          try {
            var theme = localStorage.getItem('theme') || 'light';
            document.documentElement.className = theme;
          } catch (e) {}
        })();
      `,
      tagPosition: 'head'
    }
  ]
})
</script>
```

The inline script executes synchronously before showing the element, ensuring the DOM already has the correct value. No flickering, no hydration mismatch.

This pattern is especially useful for theme toggles, user preferences, authentication states, and any client-only data that should render immediately without flashing default values.

Reference: [Nuxt 3 ClientOnly](https://nuxt.com/docs/api/components/client-only)
