---
name: elysiajs-best-practices
description: ElysiaJS performance optimization and best practices guide. Use when writing, reviewing, or refactoring ElysiaJS/Bun applications to ensure optimal performance patterns. Triggers on tasks involving ElysiaJS routes, plugins, validation, lifecycle hooks, or performance improvements.
---

# ElysiaJS Best Practices

## Overview

Comprehensive performance optimization guide for ElysiaJS and Bun applications, containing 36+ rules across 8 categories. Rules are prioritized by impact to guide automated refactoring and code generation.

ElysiaJS is a high-performance TypeScript framework built for Bun that leverages static code analysis (Sucrose) and AOT compilation to achieve near-native performance, often outperforming Node.js frameworks by 2-10×.

## When to Apply

Reference these guidelines when:
- Writing new ElysiaJS routes or plugins
- Implementing validation schemas with TypeBox
- Configuring lifecycle hooks and middleware
- Reviewing code for performance issues
- Refactoring existing ElysiaJS applications
- Optimizing API response times or memory usage

## Priority-Ordered Guidelines

Rules are prioritized by impact:

| Priority | Category | Impact |
|----------|----------|--------|
| 1 | Schema & Validation | CRITICAL |
| 2 | Plugin Architecture | CRITICAL |
| 3 | Lifecycle Optimization | HIGH |
| 4 | Async Patterns | HIGH |
| 5 | Error Handling | MEDIUM-HIGH |
| 6 | Response Optimization | MEDIUM |
| 7 | Memory & Performance | MEDIUM |
| 8 | Advanced Patterns | LOW-MEDIUM |

## Quick Reference

### Critical Patterns (Apply First)

**Schema & Validation:**
- Use TypeBox (Elysia.t) for 18× faster validation vs Zod
- Define schemas inline for better type inference
- Use `t.Numeric()` for numeric strings
- Leverage single schema for validation + types + OpenAPI
- Validate at boundaries only (request/response)

**Plugin Architecture:**
- Name plugins for automatic deduplication
- Use scoped plugins for feature isolation
- Apply `as: 'global'` only for cross-cutting concerns
- Prefer explicit dependencies over global state
- Chain methods for proper type inference

### High-Impact Lifecycle Patterns

- Order matters: hooks only apply to routes after declaration
- Use `derive` for adding validated data properties
- Use `resolve` for post-validation transformations
- Apply `guard` for multiple routes with shared schema
- Use `onBeforeHandle` to short-circuit requests

### Async & Performance Patterns

- Use `Promise.all()` for parallel operations
- Use `better-all` for dependency-based parallelization (2-10× improvement)
- Defer `await` until data is needed
- Stream responses with generator functions
- Use Server-Sent Events for real-time updates
- Enable compression for large responses

### Error Handling

- Use `onError` lifecycle for centralized handling
- Provide custom error messages via schema `error` property
- Handle `VALIDATION` errors specifically
- Return structured error responses
- Avoid leaking sensitive information

### Response Optimization

- Return typed responses for compile-time optimization
- Use streaming for large datasets
- Enable static file caching when appropriate
- Use LRU cache for cross-request caching
- Leverage Bun's native performance

## References

Full documentation with code examples is available in:

- `references/elysiajs-performance-guidelines.md` - Complete guide with all patterns
- `references/rules/` - Individual rule files organized by category

To look up a specific pattern, grep the rules directory:
```
grep -l "validation" references/rules/
grep -l "plugin" references/rules/
grep -l "guard" references/rules/
```

## Rule Categories in `references/rules/`

- `schema-*` - Schema validation and TypeBox patterns
- `plugin-*` - Plugin architecture and composition
- `lifecycle-*` - Lifecycle hooks and middleware
- `async-*` - Async patterns and parallelization
- `error-*` - Error handling patterns
- `response-*` - Response optimization
- `perf-*` - Memory and performance optimization
- `advanced-*` - Advanced patterns and macros
