---
name: hono-best-practices
description: Hono and Bun performance optimization guidelines for building ultra-fast web applications. This skill should be used when writing, reviewing, or refactoring Hono applications to ensure optimal performance patterns. Triggers on tasks involving Hono APIs, Bun runtime optimization, middleware composition, async operations, or performance improvements.
---

# Hono Best Practices

## Overview

Comprehensive performance optimization guide for Hono framework running on Bun runtime, containing 25+ rules across 7 categories. Rules are prioritized by impact to guide automated refactoring and code generation.

Hono is a small, simple, and ultrafast web framework built on Web Standards. Combined with Bun's JavaScriptCore engine and native optimizations, this stack delivers performance comparable to Rust frameworks while maintaining JavaScript/TypeScript ergonomics.

## When to Apply

Reference these guidelines when:
- Building new Hono applications or APIs
- Implementing middleware or route handlers
- Optimizing async operations and parallelization
- Optimizing response times and throughput
- Reviewing code for performance issues
- Migrating from Express, Fastify, or other frameworks

## Priority-Ordered Guidelines

Rules are prioritized by impact:

| Priority | Category | Impact |
|----------|----------|--------|
| 1 | Server Performance | CRITICAL |
| 2 | Async Operations | CRITICAL |
| 3 | Routing & Middleware | HIGH |
| 4 | Response Optimization | HIGH |
| 5 | Validation & Type Safety | MEDIUM-HIGH |
| 6 | Static Files & Assets | MEDIUM |
| 7 | Development Patterns | LOW-MEDIUM |

## Quick Reference

### Critical Patterns (Apply First)

**Server Performance:**
- Use Bun.serve() instead of Node.js adapters
- Enable HTTP/2 and connection reuse
- Leverage Bun's native streaming for large responses
- Use worker threads for CPU-intensive tasks
- Set appropriate concurrency limits

**Async Operations:**
- Use better-all for dependency-based parallelization
- Parallelize independent async operations
- Start promises early, await late
- Avoid sequential awaits for independent operations
- Use Promise.all() for truly parallel operations

### High-Impact Routing & Middleware

- Order middleware by execution frequency (guards first)
- Use specific routes before wildcard patterns
- Combine middleware to reduce chain length
- Avoid unnecessary middleware on static routes
- Use RegExpRouter (fastest in JavaScript)

### High-Impact Response Patterns

- Stream large responses instead of buffering
- Enable compression for text-based responses
- Set proper cache headers for static content
- Use content negotiation for API versioning
- Return early when validation fails

### Validation & Type Safety

- Use Zod validator for runtime type safety
- Compile RPC types ahead of time for IDE performance
- Validate at route boundaries, not in business logic
- Use discriminated unions for complex validation
- Cache validation schemas

### Static Files & Assets

- Use Bun.file() for serving static assets
- Enable aggressive caching for immutable assets
- Compress assets at build time
- Use content-based cache keys
- Implement Range request support

### Memory & Resource Management

- Use streams for large file operations
- Implement proper connection pooling for external databases
- Clean up resources in error handlers
- Monitor memory usage in production
- Avoid memory leaks in long-running processes

### Development Patterns

- Use context (c) efficiently - extract once, pass down
- Prefer function composition over inheritance
- Use JSX for templating (faster than string concat)
- Implement structured logging
- Use TypeScript for type safety

## References

Full documentation with code examples is available in:

- `references/hono-performance-guidelines.md` - Complete guide with all patterns
- `references/rules/` - Individual rule files organized by category

To look up a specific pattern, grep the rules directory:
```
grep -l "async" references/rules/
grep -l "middleware" references/rules/
grep -l "stream" references/rules/
```

## Rule Categories in `references/rules/`

- `server-*` - Bun.serve and HTTP server optimization
- `async-*` - Async operations and parallelization
- `routing-*` - Route and middleware patterns
- `response-*` - Response optimization techniques
- `validation-*` - Validation and type safety
- `static-*` - Static file serving
- `dev-*` - Development patterns and tooling

## Performance Benchmarks

Hono + Bun typically achieves:
- **Router**: Fastest in JavaScript (RegExpRouter)
- **Startup**: 4x faster than Node.js on Linux
- **Response time**: 157ms mean in production workloads
- **Bundle size**: Under 12KB with hono/tiny preset
- **Throughput**: Competes with Rust frameworks in benchmarks

## Key Differences from Node.js Frameworks

1. **No transpilation overhead** - Bun natively supports TypeScript
2. **Native HTTP server** - No need for separate HTTP library
3. **Web Standard APIs** - Fetch, Request, Response, Headers
4. **Zero dependencies** - Hono uses only Web Standard APIs
5. **Fast startup** - 4x faster cold starts than Node.js
