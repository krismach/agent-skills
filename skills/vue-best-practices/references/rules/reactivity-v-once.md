---
title: Use v-once for Static Content
impact: MEDIUM
impactDescription: skips reactive updates
tags: reactivity, v-once, static, optimization
---

## Use v-once for Static Content

Use `v-once` directive to render elements or components only once and skip subsequent updates.

**Incorrect (re-renders on every update):**

```vue
<script setup lang="ts">
const count = ref(0)
const STATIC_CONFIG = {
  title: 'My App',
  version: '1.0.0',
  author: 'John Doe'
}
</script>

<template>
  <div>
    <!-- Re-renders even though content never changes -->
    <header>
      <h1>{{ STATIC_CONFIG.title }}</h1>
      <p>Version: {{ STATIC_CONFIG.version }}</p>
      <p>By: {{ STATIC_CONFIG.author }}</p>
    </header>

    <main>
      <p>Count: {{ count }}</p>
    </main>
  </div>
</template>
```

**Correct (renders once, skips updates):**

```vue
<script setup lang="ts">
const count = ref(0)
const STATIC_CONFIG = {
  title: 'My App',
  version: '1.0.0',
  author: 'John Doe'
}
</script>

<template>
  <div>
    <!-- Renders once, never updates -->
    <header v-once>
      <h1>{{ STATIC_CONFIG.title }}</h1>
      <p>Version: {{ STATIC_CONFIG.version }}</p>
      <p>By: {{ STATIC_CONFIG.author }}</p>
    </header>

    <main>
      <p>Count: {{ count }}</p>
    </main>
  </div>
</template>
```

**For large static lists:**

```vue
<script setup lang="ts">
const STATIC_ITEMS = Array(1000).fill(0).map((_, i) => ({
  id: i,
  name: `Item ${i}`
}))

const activeId = ref(0)
</script>

<template>
  <div>
    <!-- Large list rendered once -->
    <ul v-once>
      <li v-for="item in STATIC_ITEMS" :key="item.id">
        {{ item.name }}
      </li>
    </ul>

    <!-- Only this updates -->
    <p>Active: {{ activeId }}</p>
  </div>
</template>
```

**With components:**

```vue
<template>
  <div>
    <!-- Component rendered once with initial props -->
    <StaticBanner v-once :message="welcomeMessage" />

    <DynamicContent />
  </div>
</template>
```

**When to use:**
- Configuration displays
- Copyright notices
- Initial load messages
- Large static navigation menus
- Terms and conditions text

**When NOT to use:**
- Content that might change based on route
- User preferences
- Localized text (language might change)
- A/B test content

Reference: [Vue 3 v-once Directive](https://vuejs.org/api/built-in-directives.html#v-once)
