---
title: Subscribe to Derived Boolean State
impact: MEDIUM
impactDescription: reduces update frequency
tags: reactivity, media-query, derived-state
---

## Subscribe to Derived Boolean State

Subscribe to derived boolean state instead of continuous values to reduce update frequency.

**Incorrect (updates on every pixel change):**

```vue
<script setup lang="ts">
import { useWindowSize } from '@vueuse/core'

const { width } = useWindowSize()

// Re-renders continuously as window resizes
const isMobile = computed(() => width.value < 768)
</script>

<template>
  <nav :class="isMobile ? 'mobile' : 'desktop'">
    <!-- Content -->
  </nav>
</template>
```

**Correct (updates only when breakpoint changes):**

```vue
<script setup lang="ts">
import { useMediaQuery } from '@vueuse/core'

// Only updates when crossing the breakpoint
const isMobile = useMediaQuery('(max-width: 767px)')
</script>

<template>
  <nav :class="isMobile ? 'mobile' : 'desktop'">
    <!-- Content -->
  </nav>
</template>
```

**For scroll position:**

```vue
<script setup lang="ts">
import { useScroll } from '@vueuse/core'

// Incorrect: updates continuously
const { y } = useScroll(window)
const showBackToTop = computed(() => y.value > 500)

// Correct: updates only when threshold crossed
const showBackToTop = ref(false)
const { y } = useScroll(window, {
  onScroll: () => {
    const shouldShow = y.value > 500
    if (shouldShow !== showBackToTop.value) {
      showBackToTop.value = shouldShow
    }
  }
})
</script>
```

**Custom breakpoint composable:**

```typescript
// composables/useBreakpoint.ts
export function useBreakpoint(query: string) {
  const matches = ref(false)

  onMounted(() => {
    const mediaQuery = window.matchMedia(query)
    matches.value = mediaQuery.matches

    const handler = (e: MediaQueryListEvent) => {
      matches.value = e.matches
    }

    mediaQuery.addEventListener('change', handler)

    onUnmounted(() => {
      mediaQuery.removeEventListener('change', handler)
    })
  })

  return matches
}
```

**Multiple breakpoints:**

```vue
<script setup lang="ts">
import { useBreakpoints } from '@vueuse/core'

const breakpoints = useBreakpoints({
  mobile: 0,
  tablet: 768,
  desktop: 1024,
  wide: 1280
})

const isMobile = breakpoints.smaller('tablet')
const isDesktop = breakpoints.greaterOrEqual('desktop')
</script>
```

Reference: [VueUse useMediaQuery](https://vueuse.org/core/useMediaQuery/)
