# Hono Best Practices - Rule Categories

This document describes the organization of rules in the `references/rules/` directory.

## Category Prefixes

Rules are organized by prefix to indicate their category:

### `server-*` - Server Performance (CRITICAL)
Rules for optimizing Bun's HTTP server configuration and performance.

Examples:
- `server-use-bun-serve.md` - Use Bun's native server
- `server-connection-reuse.md` - Enable connection reuse and keep-alive

### `async-*` - Async Operations (CRITICAL)
Rules for optimizing asynchronous operations and parallelization.

Examples:
- `async-dependency-parallelization.md` - Use better-all for dependency-based parallelization

### `routing-*` - Routing & Middleware (HIGH)
Rules for route organization and middleware composition.

Examples:
- `routing-middleware-order.md` - Order middleware by execution frequency
- `routing-specific-before-wildcard.md` - Define specific routes before wildcards

### `response-*` - Response Optimization (HIGH)
Rules for optimizing HTTP responses.

Examples:
- `response-stream-large-data.md` - Stream large responses
- `response-compression.md` - Enable response compression
- `response-cache-headers.md` - Set proper cache headers

### `validation-*` - Validation & Type Safety (MEDIUM-HIGH)
Rules for runtime validation and TypeScript type optimization.

Examples:
- `validation-zod-validator.md` - Use Zod validator
- `validation-rpc-type-optimization.md` - Optimize RPC type inference

### `static-*` - Static Files & Assets (MEDIUM)
Rules for serving static files efficiently.

Examples:
- `static-bun-file.md` - Use Bun.file() for static assets

### `dev-*` - Development Patterns (LOW-MEDIUM)
Rules for code organization and development practices.

Examples:
- `dev-context-efficiency.md` - Use context efficiently
- `dev-jsx-templating.md` - Use JSX for HTML templating

## Impact Levels

- **CRITICAL**: Must apply - provides 2-10x improvements
- **HIGH**: Should apply - provides significant measurable benefits
- **MEDIUM-HIGH**: Recommended - important for larger applications
- **MEDIUM**: Beneficial - improves maintainability and performance
- **LOW-MEDIUM**: Optional - minor optimizations for specific use cases

## Creating New Rules

1. Choose appropriate prefix based on category
2. Use kebab-case for filename
3. Follow the template in `_template.md`
4. Include practical code examples
5. Specify impact level and description
6. Add relevant tags
