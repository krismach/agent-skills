---
title: Use v-if/v-else for Explicit Conditional Rendering
impact: MEDIUM
impactDescription: prevents rendering 0 or NaN
tags: rendering, conditional, v-if, falsy-values
---

## Use v-if/v-else for Explicit Conditional Rendering

Use explicit `v-if`/`v-else` instead of relying on falsy values in template expressions when the condition can be `0`, `NaN`, or other falsy values that render.

**Incorrect (renders "0" when count is 0):**

```vue
<script setup lang="ts">
const count = ref(0)
</script>

<template>
  <div>
    <span v-if="count" class="badge">{{ count }}</span>
  </div>
</template>

<!-- When count = 0, renders: <div><span>0</span></div> (if v-if uses truthy check) -->
<!-- Or might not render at all depending on Vue's evaluation -->
```

**Correct (renders nothing when count is 0):**

```vue
<script setup lang="ts">
const count = ref(0)
</script>

<template>
  <div>
    <span v-if="count > 0" class="badge">{{ count }}</span>
  </div>
</template>

<!-- When count = 0, renders: <div></div> -->
<!-- When count = 5, renders: <div><span class="badge">5</span></div> -->
```

**With v-else for clarity:**

```vue
<template>
  <div>
    <span v-if="count > 0" class="badge">{{ count }}</span>
    <span v-else class="badge-empty">No items</span>
  </div>
</template>
```

**For arrays:**

```vue
<script setup lang="ts">
const items = ref<Item[]>([])
</script>

<template>
  <!-- Explicit length check -->
  <div v-if="items.length > 0">
    <div v-for="item in items" :key="item.id">{{ item.name }}</div>
  </div>
  <div v-else>No items found</div>
</template>
```

Reference: [Vue 3 Conditional Rendering](https://vuejs.org/guide/essentials/conditional.html)
