---
title: Limit SVG Path Precision
impact: LOW
impactDescription: reduces DOM size
tags: rendering, svg, optimization, precision
---

## Limit SVG Path Precision

Reduce decimal precision in SVG paths to decrease file size and DOM parsing time.

**Incorrect (excessive precision):**

```vue
<template>
  <svg viewBox="0 0 100 100">
    <path d="M 10.123456789 20.987654321 L 30.456789123 40.321654987 Q 50.789123456 60.654987321 70.123456789 80.987654321" />
  </svg>
</template>
```

**Correct (reasonable precision):**

```vue
<template>
  <svg viewBox="0 0 100 100">
    <path d="M 10.12 20.99 L 30.46 40.32 Q 50.79 60.65 70.12 80.99" />
  </svg>
</template>
```

**For complex paths, use optimization tools:**

```bash
# Install SVGO
npm install -g svgo

# Optimize SVG files
svgo input.svg -o output.svg
```

**In Vue components:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{
  points: number[][]
}>()

// Round to 2 decimal places
const pathData = computed(() => {
  return props.points
    .map(([x, y], i) => {
      const cmd = i === 0 ? 'M' : 'L'
      return `${cmd} ${x.toFixed(2)} ${y.toFixed(2)}`
    })
    .join(' ')
})
</script>

<template>
  <svg viewBox="0 0 100 100">
    <path :d="pathData" />
  </svg>
</template>
```

**Impact:**
- 10-30% reduction in SVG file size
- Faster DOM parsing
- Reduced memory usage
- No visible quality loss at normal zoom levels

Reference: [SVGO](https://github.com/svg/svgo)
