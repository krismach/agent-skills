# Sections

This file defines all sections, their ordering, impact levels, and descriptions.
The section ID (in parentheses) is the filename prefix used to group rules.

---

## 1. Schema & Validation (schema)

**Impact:** CRITICAL
**Description:** Proper schema design and validation patterns are the foundation of ElysiaJS performance. TypeBox provides 18× faster validation than alternatives and enables type inference, eliminating redundant type definitions.

## 2. Plugin Architecture (plugin)

**Impact:** CRITICAL
**Description:** Plugins are the core building blocks of ElysiaJS applications. Proper plugin composition, scoping, and dependency management directly impact maintainability and performance through deduplication and encapsulation.

## 3. Lifecycle Optimization (lifecycle)

**Impact:** HIGH
**Description:** Understanding lifecycle hooks and their execution order is essential for optimal request processing. Proper lifecycle usage eliminates redundant operations and enables request short-circuiting.

## 4. Async Patterns (async)

**Impact:** HIGH
**Description:** Async operation patterns determine request latency. Parallel execution and lazy evaluation can reduce response times by 2-10×, while poor patterns create waterfalls.

## 5. Error Handling (error)

**Impact:** MEDIUM-HIGH
**Description:** Proper error handling improves debugging, user experience, and prevents information leakage. Centralized error handling reduces code duplication and ensures consistent responses.

## 6. Response Optimization (response)

**Impact:** MEDIUM
**Description:** Response optimization techniques including streaming, compression, and caching reduce bandwidth usage and improve perceived performance.

## 7. Memory & Performance (perf)

**Impact:** MEDIUM
**Description:** Memory optimization and performance patterns that leverage Bun's capabilities and Elysia's code generation for optimal runtime characteristics.

## 8. Advanced Patterns (advanced)

**Impact:** LOW-MEDIUM
**Description:** Advanced patterns including macros, decorators, and tracing for specialized use cases that require careful implementation.
