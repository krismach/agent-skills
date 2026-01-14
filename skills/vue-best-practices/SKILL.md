---
name: vue-best-practices
description: Vue 3 and Nuxt 3 performance optimization guidelines from Vercel Engineering. This skill should be used when writing, reviewing, or refactoring Vue/Nuxt code to ensure optimal performance patterns. Triggers on tasks involving Vue components, Nuxt pages, data fetching, bundle optimization, or performance improvements.
---

# Vue Best Practices

## Overview

Comprehensive performance optimization guide for Vue 3 and Nuxt 3 applications, containing 40+ rules across 8 categories. Rules are prioritized by impact to guide automated refactoring and code generation.

## When to Apply

Reference these guidelines when:
- Writing new Vue components or Nuxt pages
- Implementing data fetching (client or server-side)
- Reviewing code for performance issues
- Refactoring existing Vue/Nuxt code
- Optimizing bundle size or load times

## Priority-Ordered Guidelines

Rules are prioritized by impact:

| Priority | Category | Impact |
|----------|----------|--------|
| 1 | Eliminating Waterfalls | CRITICAL |
| 2 | Bundle Size Optimization | CRITICAL |
| 3 | Server-Side Performance | HIGH |
| 4 | Client-Side Data Fetching | MEDIUM-HIGH |
| 5 | Reactivity Optimization | MEDIUM |
| 6 | Rendering Performance | MEDIUM |
| 7 | JavaScript Performance | LOW-MEDIUM |
| 8 | Advanced Patterns | LOW |

## Quick Reference

### Critical Patterns (Apply First)

**Eliminate Waterfalls:**
- Defer await until needed (move into branches)
- Use `Promise.all()` for independent async operations
- Use `better-all` for automatic dependency-based parallelization
- Start promises early, await late
- Use `Promise.allSettled()` for partial failures
- Use Suspense with async components to stream content

**Reduce Bundle Size:**
- Avoid barrel file imports (import directly from source)
- Use `defineAsyncComponent()` for heavy components
- Defer non-critical third-party libraries
- Preload based on user intent

### High-Impact Server Patterns

- Use `cachedFunction()` for per-request deduplication (Nuxt)
- Use `defineCachedEventHandler()` for cross-request caching
- Minimize serialization at SSR boundaries
- Parallelize data fetching with composables

### Medium-Impact Client Patterns

- Use `useFetch()` for automatic request deduplication (Nuxt)
- Defer reactive reads to usage point
- Use computed properties for derived state
- Apply shallowRef/shallowReactive for large data structures
- Use `v-once` for static content

### Rendering Patterns

- Animate wrapper elements, not SVG elements directly
- Use `v-show` instead of `v-if` for frequently toggled elements
- Prevent hydration mismatch with ClientOnly component
- Use explicit conditional rendering with `v-if`/`v-else`

### JavaScript Patterns

- Batch DOM CSS changes via classes
- Build index maps for repeated lookups
- Cache repeated function calls with computed properties
- Use `toSorted()` instead of `sort()` for immutability
- Early length check for array comparisons

## References

Full documentation with code examples is available in:

- `references/vue-performance-guidelines.md` - Complete guide with all patterns
- `references/rules/` - Individual rule files organized by category

To look up a specific pattern, grep the rules directory:
```
grep -l "suspense" references/rules/
grep -l "barrel" references/rules/
grep -l "useFetch" references/rules/
```

## Rule Categories in `references/rules/`

- `async-*` - Waterfall elimination patterns
- `bundle-*` - Bundle size optimization
- `server-*` - Server-side performance
- `client-*` - Client-side data fetching
- `reactivity-*` - Reactivity optimization
- `rendering-*` - DOM rendering performance
- `js-*` - JavaScript micro-optimizations
- `advanced-*` - Advanced patterns
