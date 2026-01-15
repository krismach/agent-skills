---
name: fastify-best-practices
description: Fastify performance optimization and best practices for Node.js applications. This skill should be used when writing, reviewing, or refactoring Fastify code to ensure optimal performance patterns. Triggers on tasks involving Fastify routes, plugins, hooks, schema validation, error handling, or performance improvements.
---

# Fastify Best Practices

## Overview

Comprehensive performance optimization guide for Fastify applications, containing 36+ rules across 7 categories. Rules are prioritized by impact to guide automated refactoring and code generation.

## When to Apply

Reference these guidelines when:
- Writing new Fastify routes or plugins
- Implementing async handlers and hooks
- Reviewing code for performance issues
- Refactoring existing Fastify code
- Optimizing request/response handling
- Working with schema validation and serialization

## Priority-Ordered Guidelines

Rules are prioritized by impact:

| Priority | Category | Impact |
|----------|----------|--------|
| 1 | Async & Request Lifecycle | CRITICAL |
| 2 | Schema Validation & Serialization | CRITICAL |
| 3 | Plugin Architecture | HIGH |
| 4 | Error Handling | HIGH |
| 5 | Route Optimization | MEDIUM-HIGH |
| 6 | Logging & Monitoring | MEDIUM |
| 7 | Deployment & Infrastructure | MEDIUM |

## Quick Reference

### Critical Patterns (Apply First)

**Async & Request Lifecycle:**
- Return data instead of calling `reply.send()` in async handlers
- Use lifecycle hooks efficiently (onRequest, preHandler, etc.)
- Avoid mixing callbacks and promises
- Always return reply when sending asynchronously
- Use async/await over callbacks
- Use `better-all` for dependency-based parallelization (2-10× faster)

**Schema Validation & Serialization:**
- Define JSON Schema for all routes
- Use serialization schemas for 100-400% performance boost
- Share schemas with `addSchema()` API
- Use AJV v8 for validation
- Enable schema caching

### High-Impact Patterns

**Plugin Architecture:**
- Use `fastify-plugin` to break encapsulation when needed
- Leverage encapsulation for modularity
- Register decorators before they're used
- Use plugins for code reuse
- Keep plugins focused and single-purpose

**Error Handling:**
- Let Fastify handle async errors automatically
- Use `setErrorHandler` for custom error handling
- Don't call `reply.send()` after errors in async handlers
- Return appropriate HTTP status codes
- Log errors with context

### Medium-Impact Patterns

**Route Optimization:**
- Use radix tree routing efficiently
- Avoid dynamic route conflicts
- Precompile schema validators
- Use decorators for shared utilities
- Minimize middleware overhead

**Logging:**
- Use Pino logger (built-in)
- Avoid synchronous logging in hot paths
- Log with appropriate levels
- Include request IDs for tracing
- Use child loggers for context

**Deployment:**
- Use reverse proxy (Nginx/HAProxy)
- Allocate 2 vCPU for low latency, 1 vCPU for high throughput
- Listen on `0.0.0.0` in containers
- Configure readiness probes correctly
- Use performance testing tools (k6, autocannon)

## References

Full documentation with code examples is available in:

- `references/fastify-performance-guidelines.md` - Complete guide with all patterns
- `references/rules/` - Individual rule files organized by category

To look up a specific pattern, grep the rules directory:
```
grep -l "async" references/rules/
grep -l "schema" references/rules/
grep -l "plugin" references/rules/
```

## Rule Categories in `references/rules/`

- `async-*` - Async lifecycle and request handling
- `schema-*` - Schema validation and serialization
- `plugin-*` - Plugin architecture patterns
- `error-*` - Error handling patterns
- `route-*` - Route optimization
- `logging-*` - Logging and monitoring
- `deploy-*` - Deployment and infrastructure
