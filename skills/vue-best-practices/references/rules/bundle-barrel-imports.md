---
title: Avoid Barrel File Imports
impact: CRITICAL
impactDescription: 200-800ms import cost, slow builds
tags: bundle, imports, tree-shaking, barrel-files, performance
---

## Avoid Barrel File Imports

Import directly from source files instead of barrel files to avoid loading thousands of unused modules. **Barrel files** are entry points that re-export multiple modules (e.g., `index.js` that does `export * from './module'`).

Popular icon and component libraries can have **up to 10,000 re-exports** in their entry file. For many packages, **it takes 200-800ms just to import them**, affecting both development speed and production cold starts.

**Why tree-shaking doesn't help:** When a library is marked as external (not bundled), the bundler can't optimize it. If you bundle it to enable tree-shaking, builds become substantially slower analyzing the entire module graph.

**Incorrect (imports entire library):**

```typescript
import { Check, X, Menu } from 'lucide-vue-next'
// Loads 1,583 modules, takes ~2.8s extra in dev
// Runtime cost: 200-800ms on every cold start

import { ElButton, ElInput } from 'element-plus'
// Loads thousands of modules, takes ~3.5s extra in dev
```

**Correct (imports only what you need):**

```typescript
import Check from 'lucide-vue-next/dist/esm/icons/check'
import X from 'lucide-vue-next/dist/esm/icons/x'
import Menu from 'lucide-vue-next/dist/esm/icons/menu'
// Loads only 3 modules (~2KB vs ~1MB)

import ElButton from 'element-plus/es/components/button'
import ElInput from 'element-plus/es/components/input'
// Loads only what you use
```

**Alternative (Nuxt 3 auto-imports):**

```typescript
// nuxt.config.ts - configure auto-imports
export default defineNuxtConfig({
  imports: {
    dirs: ['composables', 'utils']
  }
})

// Or manually configure specific imports
export default defineNuxtConfig({
  build: {
    transpile: ['element-plus']
  },
  vite: {
    optimizeDeps: {
      include: ['element-plus']
    }
  }
})
```

Direct imports provide 15-70% faster dev boot, 28% faster builds, 40% faster cold starts, and significantly faster HMR.

Libraries commonly affected: `lucide-vue-next`, `element-plus`, `ant-design-vue`, `naive-ui`, `@vueuse/core`, `lodash-es`, `date-fns`, `rxjs`.

Reference: [Vite Dependency Pre-Bundling](https://vitejs.dev/guide/dep-pre-bundling.html)
