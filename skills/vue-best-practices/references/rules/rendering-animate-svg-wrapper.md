---
title: Animate SVG Wrappers, Not SVG Elements
impact: LOW
impactDescription: smoother animations
tags: rendering, svg, animation, performance
---

## Animate SVG Wrappers, Not SVG Elements

Animate wrapper elements instead of SVG elements directly for better performance.

**Incorrect (animates SVG element):**

```vue
<template>
  <svg class="icon animate-spin">
    <path d="..." />
  </svg>
</template>

<style>
.animate-spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
</style>
```

**Correct (animates wrapper element):**

```vue
<template>
  <div class="icon-wrapper animate-spin">
    <svg class="icon">
      <path d="..." />
    </svg>
  </div>
</template>

<style>
.animate-spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
</style>
```

**Why this matters:**
- SVG transforms can trigger more expensive rendering
- Wrapper animations are hardware-accelerated more reliably
- Separates concerns between layout and graphics

**For loading spinners:**

```vue
<script setup lang="ts">
const loading = ref(true)
</script>

<template>
  <div v-if="loading" class="spinner-wrapper">
    <svg viewBox="0 0 24 24" class="spinner-icon">
      <circle cx="12" cy="12" r="10" />
    </svg>
  </div>
</template>

<style scoped>
.spinner-wrapper {
  display: inline-block;
  animation: rotate 1s linear infinite;
}

@keyframes rotate {
  to { transform: rotate(360deg); }
}

.spinner-icon {
  width: 24px;
  height: 24px;
}
</style>
```

Reference: [MDN CSS Transforms Performance](https://developer.mozilla.org/en-US/docs/Web/Performance/CSS_JavaScript_animation_performance)
