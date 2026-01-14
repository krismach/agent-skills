---
title: CSS content-visibility for Long Lists
impact: MEDIUM
impactDescription: 10× faster initial render
tags: rendering, css, content-visibility, long-lists
---

## CSS content-visibility for Long Lists

Apply `content-visibility: auto` to defer off-screen rendering.

**CSS:**

```css
.message-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;
}
```

**Example:**

```vue
<script setup lang="ts">
const messages = ref<Message[]>([])
</script>

<template>
  <div class="overflow-y-auto h-screen">
    <div
      v-for="msg in messages"
      :key="msg.id"
      class="message-item"
    >
      <Avatar :user="msg.author" />
      <div>{{ msg.content }}</div>
    </div>
  </div>
</template>

<style scoped>
.message-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;
}
</style>
```

For 1000 messages, browser skips layout/paint for ~990 off-screen items (10× faster initial render).

**Alternative with virtual scrolling:**

```vue
<script setup lang="ts">
import { useVirtualList } from '@vueuse/core'

const messages = ref<Message[]>([])
const { list, containerProps, wrapperProps } = useVirtualList(
  messages,
  { itemHeight: 80 }
)
</script>

<template>
  <div v-bind="containerProps" class="h-screen overflow-auto">
    <div v-bind="wrapperProps">
      <div
        v-for="{ data, index } in list"
        :key="index"
        class="message-item"
      >
        <Avatar :user="data.author" />
        <div>{{ data.content }}</div>
      </div>
    </div>
  </div>
</template>
```

Virtual scrolling is more performant for very large lists (10,000+ items) but requires more setup.

Reference: [VueUse useVirtualList](https://vueuse.org/core/useVirtualList/)
