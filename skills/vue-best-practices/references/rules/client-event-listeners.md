---
title: Deduplicate Global Event Listeners
impact: MEDIUM-HIGH
impactDescription: single listener for N components
tags: client, event-listeners, subscription, composables
---

## Deduplicate Global Event Listeners

Create a shared composable to deduplicate global event listeners across component instances.

**Incorrect (N instances = N listeners):**

```typescript
export function useKeyboardShortcut(key: string, callback: () => void) {
  onMounted(() => {
    const handler = (e: KeyboardEvent) => {
      if (e.metaKey && e.key === key) {
        callback()
      }
    }
    window.addEventListener('keydown', handler)

    onUnmounted(() => {
      window.removeEventListener('keydown', handler)
    })
  })
}
```

When using the `useKeyboardShortcut` composable multiple times, each instance will register a new listener.

**Correct (N instances = 1 listener):**

```typescript
// composables/useKeyboardShortcut.ts

// Module-level Map to track callbacks per key
const keyCallbacks = new Map<string, Set<() => void>>()
let listenerAttached = false

function attachGlobalListener() {
  if (listenerAttached) return

  const handler = (e: KeyboardEvent) => {
    if (e.metaKey && keyCallbacks.has(e.key)) {
      keyCallbacks.get(e.key)!.forEach(cb => cb())
    }
  }

  window.addEventListener('keydown', handler)
  listenerAttached = true

  // Cleanup when last callback is removed
  onBeforeUnmount(() => {
    if (keyCallbacks.size === 0) {
      window.removeEventListener('keydown', handler)
      listenerAttached = false
    }
  })
}

export function useKeyboardShortcut(key: string, callback: () => void) {
  onMounted(() => {
    // Register this callback
    if (!keyCallbacks.has(key)) {
      keyCallbacks.set(key, new Set())
    }
    keyCallbacks.get(key)!.add(callback)

    // Ensure global listener is attached
    attachGlobalListener()
  })

  onUnmounted(() => {
    // Unregister this callback
    const set = keyCallbacks.get(key)
    if (set) {
      set.delete(callback)
      if (set.size === 0) {
        keyCallbacks.delete(key)
      }
    }
  })
}
```

**Alternative with VueUse:**

```typescript
import { useMagicKeys } from '@vueuse/core'

export function useKeyboardShortcut(key: string, callback: () => void) {
  const keys = useMagicKeys()
  const cmdKey = keys[`Cmd+${key}`]

  watch(cmdKey, (pressed) => {
    if (pressed) {
      callback()
    }
  })
}
```

VueUse automatically deduplicates event listeners for you.

**Usage:**

```vue
<script setup lang="ts">
// Multiple shortcuts will share the same listener
useKeyboardShortcut('p', () => console.log('Profile'))
useKeyboardShortcut('k', () => console.log('Command'))
</script>
```

Reference: [VueUse useMagicKeys](https://vueuse.org/core/useMagicKeys/)
