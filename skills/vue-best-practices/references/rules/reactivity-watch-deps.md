---
title: Specify Watch Dependencies Explicitly
impact: MEDIUM
impactDescription: reduces unnecessary watcher runs
tags: reactivity, watch, dependencies
---

## Specify Watch Dependencies Explicitly

Use explicit watch sources instead of `watchEffect` to avoid tracking unnecessary dependencies.

**Incorrect (tracks all accessed reactive values):**

```vue
<script setup lang="ts">
const user = ref<User>()
const posts = ref<Post[]>([])
const comments = ref<Comment[]>([])

// Runs whenever user, posts, OR comments change
watchEffect(async () => {
  if (user.value?.id) {
    // Only needs to run when user changes
    await fetchUserData(user.value.id)
  }
  console.log(posts.value.length) // Accidentally tracked
  console.log(comments.value.length) // Accidentally tracked
})
</script>
```

**Correct (only tracks specified sources):**

```vue
<script setup lang="ts">
const user = ref<User>()
const posts = ref<Post[]>([])
const comments = ref<Comment[]>([])

// Only runs when user changes
watch(user, async (newUser) => {
  if (newUser?.id) {
    await fetchUserData(newUser.id)
  }
})

// Access posts and comments without triggering watch
console.log(posts.value.length)
console.log(comments.value.length)
</script>
```

**Watch multiple specific sources:**

```vue
<script setup lang="ts">
const firstName = ref('')
const lastName = ref('')
const email = ref('')

// Only runs when firstName or lastName change, not email
watch([firstName, lastName], ([first, last]) => {
  console.log(`Name: ${first} ${last}`)
})
</script>
```

**Use watchEffect only for truly dependent logic:**

```vue
<script setup lang="ts">
const x = ref(0)
const y = ref(0)

// Appropriate: needs to track both x and y
watchEffect(() => {
  console.log(`Position: ${x.value}, ${y.value}`)
})
</script>
```

Reference: [Vue 3 Watchers](https://vuejs.org/guide/essentials/watchers.html)
