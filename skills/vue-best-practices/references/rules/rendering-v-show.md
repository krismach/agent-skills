---
title: Use v-show for Frequently Toggled Elements
impact: MEDIUM
impactDescription: avoids re-render cost
tags: rendering, v-show, v-if, toggle
---

## Use v-show for Frequently Toggled Elements

Use `v-show` instead of `v-if` for elements that toggle frequently to avoid mounting/unmounting overhead.

**Incorrect (destroys and recreates on every toggle):**

```vue
<script setup lang="ts">
const isVisible = ref(false)

// User toggles this frequently
const toggle = () => {
  isVisible.value = !isVisible.value
}
</script>

<template>
  <div>
    <button @click="toggle">Toggle</button>
    <!-- Component destroyed and recreated each time -->
    <HeavyComponent v-if="isVisible" />
  </div>
</template>
```

**Correct (toggles CSS display property):**

```vue
<script setup lang="ts">
const isVisible = ref(false)

const toggle = () => {
  isVisible.value = !isVisible.value
}
</script>

<template>
  <div>
    <button @click="toggle">Toggle</button>
    <!-- Component stays mounted, only CSS changes -->
    <HeavyComponent v-show="isVisible" />
  </div>
</template>
```

**When to use v-if vs v-show:**

Use `v-if` when:
- Condition rarely changes
- Initial render cost matters more than toggle cost
- Need to prevent component initialization

Use `v-show` when:
- Condition toggles frequently
- Component is expensive to mount/unmount
- Preserving component state across toggles

**Example (modal/dialog):**

```vue
<script setup lang="ts">
const showModal = ref(false)
</script>

<template>
  <!-- Modal toggled frequently, use v-show -->
  <Modal v-show="showModal" @close="showModal = false">
    <ExpensiveForm />
  </Modal>
</template>
```

**Example (conditional routes):**

```vue
<script setup lang="ts">
const currentTab = ref('home')
</script>

<template>
  <!-- Tabs change frequently, use v-show -->
  <div>
    <TabContent v-show="currentTab === 'home'" />
    <TabContent v-show="currentTab === 'profile'" />
    <TabContent v-show="currentTab === 'settings'" />
  </div>
</template>
```

Reference: [Vue 3 v-show vs v-if](https://vuejs.org/guide/essentials/conditional.html#v-if-vs-v-show)
