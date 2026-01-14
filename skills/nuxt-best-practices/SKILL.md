---
name: nuxt-best-practices
description: Nuxt 3 and Vue 3 performance optimization guidelines. This skill should be used when writing, reviewing, or refactoring Nuxt 3/Vue 3 code to ensure optimal performance patterns. Triggers on tasks involving Vue components, Nuxt pages, data fetching, bundle optimization, or performance improvements.
---

# Nuxt Best Practices

## Overview

Comprehensive performance optimization guide for Nuxt 3 and Vue 3 applications, containing 40+ rules across 8 categories. Rules are prioritized by impact to guide automated refactoring and code generation.

## When to Apply

Reference these guidelines when:
- Writing new Vue components or Nuxt pages
- Implementing data fetching (client or server-side)
- Reviewing code for performance issues
- Refactoring existing Nuxt/Vue code
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
- Use parallel data fetching with Promise.all()
- Defer await until needed (move into branches)
- Use parallel composables (useFetch, useAsyncData simultaneously)
- Use `better-all` for dependency-based parallelization
- Avoid sequential API calls in composables
- Stream content with Suspense and lazy components

**Reduce Bundle Size:**
- Use auto-imports effectively (avoid manual imports)
- Use lazy loading for heavy components (defineAsyncComponent)
- Defer non-critical third-party libraries
- Tree-shake unused Nuxt modules
- Optimize icon imports (use individual imports)

### High-Impact Server Patterns

- Use cached composables for server-side data
- Minimize serialization at hydration boundaries
- Use nitro caching for expensive operations
- Parallelize data fetching with component composition
- Leverage Nuxt's route rules for static generation

### Medium-Impact Client Patterns

- Use computed properties instead of methods
- Avoid unnecessary watchers (prefer computed)
- Use keepalive for expensive components
- Defer state reads to usage point
- Use shallow refs for large objects

### Rendering Patterns

- Animate wrapper divs, not SVG elements
- Use virtual scrolling for long lists
- Prevent hydration mismatch with client-only
- Use explicit v-if instead of v-show for heavy components
- Use CSS containment for isolated components

### JavaScript Patterns

- Batch DOM changes via CSS classes
- Build index maps for repeated lookups
- Cache repeated function calls
- Use toSorted() instead of sort() for immutability
- Early length check for array comparisons

## References

Full documentation with code examples is available in:

- `references/nuxt-performance-guidelines.md` - Complete guide with all patterns
- `references/rules/` - Individual rule files organized by category

To look up a specific pattern, grep the rules directory:
```
grep -l "useFetch" references/rules/
grep -l "lazy" references/rules/
grep -l "cache" references/rules/
```

## Rule Categories in `references/rules/`

- `async-*` - Waterfall elimination patterns
- `bundle-*` - Bundle size optimization
- `server-*` - Server-side performance (Nitro)
- `client-*` - Client-side data fetching
- `reactivity-*` - Vue reactivity optimization
- `rendering-*` - DOM rendering performance
- `js-*` - JavaScript micro-optimizations
- `advanced-*` - Advanced patterns
