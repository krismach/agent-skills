---
title: Conditional Imports for Platform-Specific Code
impact: MEDIUM
impactDescription: reduces bundle size
tags: bundle, conditional, tree-shaking
---

## Conditional Imports for Platform-Specific Code

Use conditional imports to avoid bundling platform-specific code that won't be used.

**Incorrect (bundles both client and server code):**

```typescript
import { clientOnlyFunction } from './client-utils'
import { serverOnlyFunction } from './server-utils'

export function hybridFunction() {
  if (import.meta.client) {
    return clientOnlyFunction()
  } else {
    return serverOnlyFunction()
  }
}
```

**Correct (only bundles what's needed):**

```typescript
export async function hybridFunction() {
  if (import.meta.client) {
    const { clientOnlyFunction } = await import('./client-utils')
    return clientOnlyFunction()
  } else {
    const { serverOnlyFunction } = await import('./server-utils')
    return serverOnlyFunction()
  }
}
```

**Better with static analysis:**

```typescript
// client-utils.ts
export function clientOnlyFunction() {
  // Client-only code
}

// server-utils.ts
export function serverOnlyFunction() {
  // Server-only code
}

// Use separate entry points
// composables/useHybrid.client.ts
export function useHybrid() {
  return clientOnlyFunction()
}

// composables/useHybrid.server.ts
export function useHybrid() {
  return serverOnlyFunction()
}
```

**In Nuxt 3, use .client and .server suffixes:**

```
composables/
  useAnalytics.client.ts  # Only bundled for client
  useDatabase.server.ts   # Only bundled for server
```

Nuxt automatically excludes `.server.ts` files from client bundle and `.client.ts` files from server bundle.

Reference: [Nuxt 3 Server and Client Components](https://nuxt.com/docs/guide/directory-structure/components#client-components)
