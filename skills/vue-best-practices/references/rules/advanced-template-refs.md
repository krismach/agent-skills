---
title: Store Event Handlers in Refs
impact: LOW
impactDescription: stable subscriptions
tags: advanced, refs, event-handlers, optimization
---

## Store Event Handlers in Refs

Store callbacks in refs when used in watchers/effects that shouldn't re-run on callback changes.

**Incorrect (re-subscribes on every callback change):**

```typescript
export function useWindowEvent(event: string, handler: () => void) {
  onMounted(() => {
    window.addEventListener(event, handler)

    onUnmounted(() => {
      window.removeEventListener(event, handler)
    })
  })

  // If handler changes, we're stuck with the old one!
}
```

**Correct (stable subscription with latest handler):**

```typescript
export function useWindowEvent(event: string, handler: () => void) {
  const handlerRef = ref(handler)

  // Keep ref updated
  watch(() => handler, (newHandler) => {
    handlerRef.value = newHandler
  })

  onMounted(() => {
    const listener = (...args: any[]) => handlerRef.value(...args)
    window.addEventListener(event, listener)

    onUnmounted(() => {
      window.removeEventListener(event, listener)
    })
  })
}
```

**Alternative with VueUse:**

```typescript
import { useEventListener } from '@vueuse/core'

// VueUse handles this automatically
export function useWindowEvent(event: string, handler: () => void) {
  useEventListener(window, event, handler)
}
```

**For custom composables:**

```typescript
export function useCustomEffect(callback: () => void, deps: Ref[]) {
  const callbackRef = ref(callback)

  // Update ref when callback changes
  watchEffect(() => {
    callbackRef.value = callback
  })

  // Watch deps, use latest callback
  watch(deps, () => {
    callbackRef.value()
  })
}
```

Reference: [VueUse useEventListener](https://vueuse.org/core/useEventListener/)
