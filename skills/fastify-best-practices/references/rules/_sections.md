# Sections

This file defines all sections, their ordering, impact levels, and descriptions.
The section ID (in parentheses) is the filename prefix used to group rules.

---

## 1. Async & Request Lifecycle (async)

**Impact:** CRITICAL
**Description:** Proper async handling is essential for Fastify performance. Mistakes in async patterns can cause silent errors, memory leaks, or incorrect responses.

## 2. Schema Validation & Serialization (schema)

**Impact:** CRITICAL
**Description:** JSON Schema validation and serialization are Fastify's performance superpowers. Using them correctly can improve throughput by 100-400%.

## 3. Plugin Architecture (plugin)

**Impact:** HIGH
**Description:** Fastify's plugin system enables modularity and code reuse while maintaining performance. Proper plugin usage is key to scalable applications.

## 4. Error Handling (error)

**Impact:** HIGH
**Description:** Proper error handling ensures reliability and debuggability. Fastify's async error handling is one of its key advantages over Express.

## 5. Route Optimization (route)

**Impact:** MEDIUM-HIGH
**Description:** Fastify's routing is one of the fastest available, but improper usage can negate these benefits.

## 6. Logging & Monitoring (logging)

**Impact:** MEDIUM
**Description:** Logging is essential for debugging and monitoring, but improper logging can significantly impact performance.

## 7. Deployment & Infrastructure (deploy)

**Impact:** MEDIUM
**Description:** Proper deployment configuration ensures Fastify runs optimally in production.
