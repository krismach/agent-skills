# Fastify Best Practices

**Version 1.0.0**
Vercel Engineering
January 2026

> **Note:**
> This document is mainly for agents and LLMs to follow when maintaining,
> generating, or refactoring Fastify codebases. Humans may also find it useful,
> but guidance here is optimized for automation and consistency by AI-assisted workflows.

---

## Abstract

Comprehensive performance optimization guide for Fastify applications, designed for AI agents and LLMs. Contains 36+ rules across 7 categories, prioritized by impact from critical (async handling, schema validation) to incremental (deployment patterns). Each rule includes detailed explanations, real-world examples comparing incorrect vs. correct implementations, and specific impact metrics to guide automated refactoring and code generation.

---

## Table of Contents

1. [Async & Request Lifecycle](#1-async--request-lifecycle) — **CRITICAL**
   - 1.1 [Return Data Instead of reply.send()](#11)
   - 1.2 [Avoid Mixing Callbacks and Promises](#12)
   - 1.3 [Always Return reply When Sending Async](#13)
   - 1.4 [Use Lifecycle Hooks Efficiently](#14)
   - 1.5 [Prefer async/await Over Callbacks](#15)
   - 1.6 [Use Dependency-Based Parallelization](#16)
2. [Schema Validation & Serialization](#2-schema-validation--serialization) — **CRITICAL**
   - 2.1 [Define JSON Schema for All Routes](#21)
   - 2.2 [Use Serialization Schemas](#22)
   - 2.3 [Share Schemas with addSchema()](#23)
   - 2.4 [Leverage Schema Caching](#24)
   - 2.5 [Use Fluent-Schema for Complex Schemas](#25)
3. [Plugin Architecture](#3-plugin-architecture) — **HIGH**
   - 3.1 [Use fastify-plugin for Shared Decorators](#31)
   - 3.2 [Leverage Encapsulation](#32)
   - 3.3 [Register Decorators Before Use](#33)
   - 3.4 [Keep Plugins Focused](#34)
   - 3.5 [Avoid Unnecessary Plugins](#35)
4. [Error Handling](#4-error-handling) — **HIGH**
   - 4.1 [Let Fastify Handle Async Errors](#41)
   - 4.2 [Use setErrorHandler for Custom Errors](#42)
   - 4.3 [Don't Send After Errors in Async](#43)
   - 4.4 [Return Appropriate Status Codes](#44)
   - 4.5 [Log Errors with Context](#45)
5. [Route Optimization](#5-route-optimization) — **MEDIUM-HIGH**
   - 5.1 [Use Radix Tree Routing Efficiently](#51)
   - 5.2 [Avoid Dynamic Route Conflicts](#52)
   - 5.3 [Precompile Validators](#53)
   - 5.4 [Use Decorators for Shared Utils](#54)
   - 5.5 [Minimize Hook Overhead](#55)
6. [Logging & Monitoring](#6-logging--monitoring) — **MEDIUM**
   - 6.1 [Use Pino Logger](#61)
   - 6.2 [Avoid Synchronous Logging](#62)
   - 6.3 [Use Appropriate Log Levels](#63)
   - 6.4 [Include Request IDs](#64)
   - 6.5 [Use Child Loggers for Context](#65)
7. [Deployment & Infrastructure](#7-deployment--infrastructure) — **MEDIUM**
   - 7.1 [Use Reverse Proxy](#71)
   - 7.2 [Optimize vCPU Allocation](#72)
   - 7.3 [Listen on 0.0.0.0 in Containers](#73)
   - 7.4 [Configure Readiness Probes](#74)
   - 7.5 [Use Performance Testing Tools](#75)

---

## 1. Async & Request Lifecycle

**Impact: CRITICAL**

Proper async handling is essential for Fastify performance. Mistakes in async patterns can cause silent errors, memory leaks, or incorrect responses.

### 1.1 Return Data Instead of reply.send()

When using async/await, return the data from the handler instead of calling `reply.send()`. This prevents silent errors and follows Fastify's recommended pattern.

**Incorrect: using reply.send() in async handler**

```javascript
fastify.get('/user/:id', async (request, reply) => {
  const user = await getUserById(request.params.id)
  reply.send(user) // Works, but not recommended
})
```

If an error occurs after `reply.send()`, it won't be caught or reported.

**Correct: return data directly**

```javascript
fastify.get('/user/:id', async (request, reply) => {
  const user = await getUserById(request.params.id)
  return user // Fastify handles serialization and errors
})
```

Fastify automatically serializes the return value and catches any errors that occur.

**Why this matters:**

```javascript
// Incorrect - error after send is silent
fastify.get('/data', async (request, reply) => {
  reply.send({ data: 'value' })
  throw new Error('This error is never reported!') // Silent failure
})

// Correct - error is caught and reported
fastify.get('/data', async (request, reply) => {
  const result = { data: 'value' }
  throw new Error('This error is caught by Fastify') // Returns 500
  return result
})
```

### 1.2 Avoid Mixing Callbacks and Promises

Never mix callback style (using `done`) with promises/async-await. The hook chain will execute twice, causing unpredictable behavior.

**Incorrect: mixing callbacks and async**

```javascript
fastify.addHook('onRequest', async (request, reply, done) => {
  await doSomethingAsync()
  done() // Hook executes twice!
})
```

**Correct: use only async/await**

```javascript
fastify.addHook('onRequest', async (request, reply) => {
  await doSomethingAsync()
  // No done() needed
})
```

**Correct: use only callbacks**

```javascript
fastify.addHook('onRequest', (request, reply, done) => {
  doSomethingAsync()
    .then(() => done())
    .catch(done)
})
```

### 1.3 Always Return reply When Sending Async

When calling `reply.send()` outside the promise chain (e.g., in `setImmediate`), you must `return reply` to prevent the request from continuing.

**Incorrect: not returning reply**

```javascript
fastify.addHook('preHandler', async (request, reply) => {
  setImmediate(() => {
    reply.send('hello')
  })
  // Request continues to handler!
})
```

**Correct: return reply to stop execution**

```javascript
fastify.addHook('preHandler', async (request, reply) => {
  setImmediate(() => {
    reply.send('hello')
  })
  return reply // Mandatory to prevent request from continuing
})
```

### 1.4 Use Lifecycle Hooks Efficiently

Choose the right hook for the right task. Understanding the execution order prevents unnecessary work.

**Hook Execution Order:**
1. `onRequest` - Before parsing (no body available)
2. `preParsing` - Before body parsing
3. `preValidation` - After parsing, before validation
4. `preHandler` - After validation, before handler
5. Handler execution
6. `preSerialization` - Before serialization
7. `onSend` - Before sending response
8. `onResponse` - After response sent
9. `onTimeout` - If request times out
10. `onError` - If error occurs

**Incorrect: using preHandler when onRequest would work**

```javascript
// Don't parse body if you don't need it
fastify.addHook('preHandler', async (request, reply) => {
  // Body is already parsed, wasting resources
  if (!request.headers.authorization) {
    throw new Error('Unauthorized')
  }
})
```

**Correct: use onRequest for early checks**

```javascript
// Check auth before parsing body
fastify.addHook('onRequest', async (request, reply) => {
  if (!request.headers.authorization) {
    throw new Error('Unauthorized')
  }
})
```

**When to use each hook:**

- `onRequest` - Auth checks, logging, early rejection
- `preHandler` - Data manipulation, additional validation
- `onSend` - Response modification, compression
- `onError` - Custom error handling, error logging

### 1.5 Prefer async/await Over Callbacks

Use async/await for cleaner, more maintainable code. Fastify fully supports async patterns.

**Incorrect: callback style**

```javascript
fastify.get('/users', (request, reply) => {
  getUsers()
    .then(users => {
      reply.send(users)
    })
    .catch(error => {
      reply.code(500).send({ error: error.message })
    })
})
```

**Correct: async/await**

```javascript
fastify.get('/users', async (request, reply) => {
  const users = await getUsers()
  return users
  // Errors are automatically caught and handled
})
```

### 1.6 Use Dependency-Based Parallelization

For operations with partial dependencies, use `better-all` to maximize parallelism. It automatically starts each task at the earliest possible moment.

**Incorrect: profile waits for config unnecessarily**

```javascript
fastify.get('/dashboard/:userId', async (request, reply) => {
  const [user, config] = await Promise.all([
    getUser(request.params.userId),
    getAppConfig()
  ])

  const [posts, profile] = await Promise.all([
    getUserPosts(user.id),
    getUserProfile(user.id)
  ])

  return { user, config, posts, profile }
})
```

In this example, `posts` and `profile` wait for both `user` AND `config` to complete, even though they only need `user`.

**Correct: config and profile run in optimal parallel**

```javascript
import { all } from 'better-all'

fastify.get('/dashboard/:userId', async (request, reply) => {
  const { user, config, posts, profile } = await all({
    async user() {
      return getUser(request.params.userId)
    },
    async config() {
      return getAppConfig()
    },
    async posts() {
      const userData = await this.$.user
      return getUserPosts(userData.id)
    },
    async profile() {
      const userData = await this.$.user
      return getUserProfile(userData.id)
    }
  })

  return { user, config, posts, profile }
})
```

With `better-all`:
- `user` and `config` start immediately (parallel)
- `posts` and `profile` start as soon as `user` completes
- `posts` and `profile` don't wait for `config` to finish

**Complex example with multiple dependency levels:**

```javascript
import { all } from 'better-all'

fastify.get('/project/:id', async (request, reply) => {
  const { session, project, permissions, analytics, team, activity } = await all({
    // Independent: starts immediately
    async session() {
      return getSession(request.headers.authorization)
    },

    // Independent: starts immediately
    async project() {
      return getProject(request.params.id)
    },

    // Depends on session: starts when session completes
    async permissions() {
      const s = await this.$.session
      return getUserPermissions(s.userId, request.params.id)
    },

    // Depends on project: starts when project completes
    async analytics() {
      const p = await this.$.project
      return getProjectAnalytics(p.id)
    },

    // Depends on project: starts when project completes (parallel with analytics)
    async team() {
      const p = await this.$.project
      return getProjectTeam(p.teamId)
    },

    // Depends on both session and project: starts when both complete
    async activity() {
      const [s, p] = await Promise.all([this.$.session, this.$.project])
      return getUserActivityOnProject(s.userId, p.id)
    }
  })

  return { project, permissions, analytics, team, activity }
})
```

**Execution timeline comparison:**

```
Sequential (500ms total):
0ms:   session starts
100ms: session done, project starts
200ms: project done, permissions starts
300ms: permissions done, analytics starts
400ms: analytics done, team starts
500ms: all done

Promise.all without dependencies (200ms total):
0ms:   session, project start
100ms: both done, permissions, analytics, team, activity start
200ms: all done

better-all with dependencies (150ms total):
0ms:   session, project start (parallel)
50ms:  session done → permissions starts
100ms: project done → analytics, team start (parallel)
100ms: both done → activity starts
150ms: all done
```

**Installation:**

```bash
npm install better-all
```

**When to use:**
- 3+ async operations with partial dependencies
- Complex route handlers with multiple data sources
- Operations where some depend on results from others

**Performance impact:** 2-10× improvement over sequential, 1.5-3× improvement over manual Promise.all chains.

Reference: [https://github.com/shuding/better-all](https://github.com/shuding/better-all)

---

## 2. Schema Validation & Serialization

**Impact: CRITICAL**

JSON Schema validation and serialization are Fastify's performance superpowers. Using them correctly can improve throughput by 100-400%.

### 2.1 Define JSON Schema for All Routes

Always define schemas for route validation. Fastify compiles these into highly performant functions using AJV v8.

**Incorrect: no schema validation**

```javascript
fastify.post('/user', async (request, reply) => {
  // Manual validation is slower and error-prone
  if (!request.body.email || typeof request.body.email !== 'string') {
    throw new Error('Invalid email')
  }
  if (!request.body.age || typeof request.body.age !== 'number') {
    throw new Error('Invalid age')
  }

  return createUser(request.body)
})
```

**Correct: use JSON Schema**

```javascript
fastify.post('/user', {
  schema: {
    body: {
      type: 'object',
      required: ['email', 'age'],
      properties: {
        email: { type: 'string', format: 'email' },
        age: { type: 'number', minimum: 0, maximum: 120 }
      }
    }
  }
}, async (request, reply) => {
  // request.body is validated - no manual checks needed
  return createUser(request.body)
})
```

**Benefits:**
- 10-100x faster than manual validation
- Automatic 400 responses on validation failure
- Built-in type coercion
- Self-documenting API

### 2.2 Use Serialization Schemas

Define response schemas for 100-400% performance boost. `fast-json-stringify` is much faster than `JSON.stringify()`.

**Incorrect: no response schema**

```javascript
fastify.get('/users', async (request, reply) => {
  const users = await getUsers()
  return users // Uses slow JSON.stringify()
})
```

**Correct: use response schema**

```javascript
fastify.get('/users', {
  schema: {
    response: {
      200: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            id: { type: 'number' },
            name: { type: 'string' },
            email: { type: 'string' }
            // Only these fields are serialized
          }
        }
      }
    }
  }
}, async (request, reply) => {
  const users = await getUsers()
  return users // Uses fast-json-stringify - 100-400% faster
})
```

**Performance comparison:**
- Without schema: `JSON.stringify()` - baseline
- With schema: `fast-json-stringify()` - 100-400% faster
- Bonus: Automatically filters unwanted properties

### 2.3 Share Schemas with addSchema()

Reuse common schemas across routes using the `addSchema()` API. This improves maintainability and enables schema composition.

**Incorrect: duplicating schemas**

```javascript
fastify.get('/user/:id', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          id: { type: 'number' },
          name: { type: 'string' },
          email: { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => { /* ... */ })

fastify.post('/user', {
  schema: {
    response: {
      201: {
        type: 'object',
        properties: {
          id: { type: 'number' },
          name: { type: 'string' },
          email: { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => { /* ... */ })
```

**Correct: share schemas**

```javascript
// Register shared schema once
fastify.addSchema({
  $id: 'user',
  type: 'object',
  properties: {
    id: { type: 'number' },
    name: { type: 'string' },
    email: { type: 'string' }
  }
})

// Reference it in routes
fastify.get('/user/:id', {
  schema: {
    response: {
      200: { $ref: 'user#' }
    }
  }
}, async (request, reply) => { /* ... */ })

fastify.post('/user', {
  schema: {
    response: {
      201: { $ref: 'user#' }
    }
  }
}, async (request, reply) => { /* ... */ })
```

### 2.4 Leverage Schema Caching

Fastify automatically caches compiled validators and serializers using WeakMap. Avoid recreating schemas unnecessarily.

**Incorrect: recreating schemas**

```javascript
function createRoute(type) {
  return {
    schema: {
      // New schema object on every call - not cached!
      body: {
        type: 'object',
        properties: {
          type: { type: 'string', enum: [type] }
        }
      }
    },
    handler: async (request, reply) => { /* ... */ }
  }
}

fastify.get('/type-a', createRoute('a'))
fastify.get('/type-b', createRoute('b'))
```

**Correct: reuse schema objects**

```javascript
const schemas = {
  a: {
    body: {
      type: 'object',
      properties: {
        type: { type: 'string', enum: ['a'] }
      }
    }
  },
  b: {
    body: {
      type: 'object',
      properties: {
        type: { type: 'string', enum: ['b'] }
      }
    }
  }
}

fastify.get('/type-a', { schema: schemas.a }, async (request, reply) => { /* ... */ })
fastify.get('/type-b', { schema: schemas.b }, async (request, reply) => { /* ... */ })
```

### 2.5 Use Fluent-Schema for Complex Schemas

For complex schemas, use `fluent-json-schema` for better readability and type safety.

**Incorrect: complex nested schemas**

```javascript
const schema = {
  body: {
    type: 'object',
    required: ['user', 'settings'],
    properties: {
      user: {
        type: 'object',
        required: ['name', 'email'],
        properties: {
          name: { type: 'string', minLength: 1 },
          email: { type: 'string', format: 'email' },
          age: { type: 'number', minimum: 0 }
        }
      },
      settings: {
        type: 'object',
        properties: {
          notifications: { type: 'boolean' },
          theme: { type: 'string', enum: ['light', 'dark'] }
        }
      }
    }
  }
}
```

**Correct: use fluent-schema**

```javascript
const S = require('fluent-json-schema')

const userSchema = S.object()
  .prop('name', S.string().minLength(1).required())
  .prop('email', S.string().format('email').required())
  .prop('age', S.number().minimum(0))

const settingsSchema = S.object()
  .prop('notifications', S.boolean())
  .prop('theme', S.string().enum(['light', 'dark']))

const schema = {
  body: S.object()
    .prop('user', userSchema.required())
    .prop('settings', settingsSchema.required())
}
```

---

## 3. Plugin Architecture

**Impact: HIGH**

Fastify's plugin system enables modularity and code reuse while maintaining performance. Proper plugin usage is key to scalable applications.

### 3.1 Use fastify-plugin for Shared Decorators

Use `fastify-plugin` when you need decorators or hooks to be available in parent scope.

**Incorrect: decorator not available to parent**

```javascript
// plugins/auth.js
async function authPlugin(fastify, options) {
  fastify.decorate('authenticate', async (request) => {
    // Auth logic
  })
}

module.exports = authPlugin

// app.js
fastify.register(authPlugin)

fastify.get('/protected', async (request, reply) => {
  await fastify.authenticate(request) // Error: authenticate is not defined
  return { data: 'secret' }
})
```

**Correct: use fastify-plugin**

```javascript
// plugins/auth.js
const fp = require('fastify-plugin')

async function authPlugin(fastify, options) {
  fastify.decorate('authenticate', async (request) => {
    // Auth logic
  })
}

module.exports = fp(authPlugin)

// app.js
fastify.register(authPlugin)

fastify.get('/protected', async (request, reply) => {
  await fastify.authenticate(request) // Works!
  return { data: 'secret' }
})
```

### 3.2 Leverage Encapsulation

Use encapsulation to create modular, isolated components with their own routes, decorators, and hooks.

**Incorrect: global plugin pollution**

```javascript
// Everything is global, causing naming conflicts
fastify.decorate('dbV1', dbConnectionV1)
fastify.decorate('dbV2', dbConnectionV2)

fastify.register(async (fastify) => {
  fastify.get('/api/v1/users', async () => {
    return fastify.dbV1.query('SELECT * FROM users')
  })
})

fastify.register(async (fastify) => {
  fastify.get('/api/v2/users', async () => {
    return fastify.dbV2.query('SELECT * FROM users')
  })
})
```

**Correct: use encapsulation**

```javascript
fastify.register(async (fastify) => {
  // db decorator only available in this context
  fastify.decorate('db', dbConnectionV1)

  fastify.get('/api/v1/users', async (request, reply) => {
    return fastify.db.query('SELECT * FROM users')
  })
}, { prefix: '/api/v1' })

fastify.register(async (fastify) => {
  // Different db decorator, no conflict
  fastify.decorate('db', dbConnectionV2)

  fastify.get('/api/v2/users', async (request, reply) => {
    return fastify.db.query('SELECT * FROM users')
  })
}, { prefix: '/api/v2' })
```

### 3.3 Register Decorators Before Use

Always register decorators before they're used. Fastify optimizes object shape at instantiation time.

**Incorrect: late decorator registration**

```javascript
fastify.get('/user/:id', async (request, reply) => {
  // Decorator doesn't exist yet - error!
  return fastify.getUserById(request.params.id)
})

// Too late!
fastify.decorate('getUserById', async (id) => {
  return db.query('SELECT * FROM users WHERE id = ?', [id])
})
```

**Correct: register before use**

```javascript
// Register decorator first
fastify.decorate('getUserById', async (id) => {
  return db.query('SELECT * FROM users WHERE id = ?', [id])
})

fastify.get('/user/:id', async (request, reply) => {
  // Decorator is available
  return fastify.getUserById(request.params.id)
})
```

**Why this matters:**

Decorating objects before they're instantiated allows the JavaScript engine to optimize the object shape (hidden class), resulting in faster property access.

### 3.4 Keep Plugins Focused

Each plugin should have a single, clear purpose. This improves testability and reusability.

**Incorrect: kitchen sink plugin**

```javascript
async function megaPlugin(fastify, options) {
  // Database connection
  const db = await connectToDatabase()
  fastify.decorate('db', db)

  // Authentication
  fastify.decorate('authenticate', async (request) => { /* ... */ })

  // Email sending
  fastify.decorate('sendEmail', async (to, subject, body) => { /* ... */ })

  // File uploads
  fastify.register(require('fastify-multipart'))

  // Routes
  fastify.get('/users', async () => { /* ... */ })
  fastify.post('/login', async () => { /* ... */ })
}
```

**Correct: focused plugins**

```javascript
// plugins/database.js
async function databasePlugin(fastify, options) {
  const db = await connectToDatabase(options)
  fastify.decorate('db', db)
}

// plugins/auth.js
async function authPlugin(fastify, options) {
  fastify.decorate('authenticate', async (request) => { /* ... */ })
}

// plugins/email.js
async function emailPlugin(fastify, options) {
  fastify.decorate('sendEmail', async (to, subject, body) => { /* ... */ })
}

// routes/users.js
async function userRoutes(fastify, options) {
  fastify.get('/users', async () => { /* ... */ })
}

// app.js
fastify.register(databasePlugin, { connection: 'postgresql://...' })
fastify.register(authPlugin)
fastify.register(emailPlugin)
fastify.register(userRoutes, { prefix: '/api' })
```

### 3.5 Avoid Unnecessary Plugins

Only use plugins when they provide real value. Each plugin adds overhead to the encapsulation context.

**Incorrect: plugin for simple utility**

```javascript
// plugins/utils.js
const fp = require('fastify-plugin')

async function utilsPlugin(fastify, options) {
  fastify.decorate('formatDate', (date) => {
    return date.toISOString()
  })
}

module.exports = fp(utilsPlugin)
```

**Correct: use simple module**

```javascript
// utils/date.js
function formatDate(date) {
  return date.toISOString()
}

module.exports = { formatDate }

// routes/users.js
const { formatDate } = require('../utils/date')

fastify.get('/users', async (request, reply) => {
  const users = await getUsers()
  return users.map(u => ({
    ...u,
    createdAt: formatDate(u.createdAt)
  }))
})
```

**When to use plugins:**
- Sharing database connections
- Adding middleware (auth, CORS, etc.)
- Registering groups of routes
- Adding decorators that need Fastify instance

**When NOT to use plugins:**
- Simple utility functions
- Business logic
- Data transformations
- Pure functions

---

## 4. Error Handling

**Impact: HIGH**

Proper error handling ensures reliability and debuggability. Fastify's async error handling is one of its key advantages over Express.

### 4.1 Let Fastify Handle Async Errors

Fastify automatically catches errors in async handlers and reports them as HTTP 500. Don't wrap everything in try-catch.

**Incorrect: unnecessary try-catch**

```javascript
fastify.get('/users', async (request, reply) => {
  try {
    const users = await getUsers()
    return users
  } catch (error) {
    reply.code(500).send({ error: error.message })
  }
})
```

**Correct: let Fastify handle it**

```javascript
fastify.get('/users', async (request, reply) => {
  const users = await getUsers()
  return users
  // Errors are automatically caught and reported
})
```

**When to use try-catch:**

```javascript
fastify.get('/user/:id', async (request, reply) => {
  try {
    const user = await getUserById(request.params.id)
    return user
  } catch (error) {
    // Only catch when you need custom handling
    if (error.code === 'USER_NOT_FOUND') {
      reply.code(404).send({ error: 'User not found' })
      return
    }
    // Re-throw for Fastify to handle
    throw error
  }
})
```

### 4.2 Use setErrorHandler for Custom Errors

Define a global error handler to customize error responses.

**Incorrect: handling errors in every route**

```javascript
fastify.get('/users', async (request, reply) => {
  try {
    const users = await getUsers()
    return users
  } catch (error) {
    request.log.error(error)
    reply.code(500).send({ error: 'Internal Server Error' })
  }
})

fastify.get('/posts', async (request, reply) => {
  try {
    const posts = await getPosts()
    return posts
  } catch (error) {
    request.log.error(error)
    reply.code(500).send({ error: 'Internal Server Error' })
  }
})
```

**Correct: use error handler**

```javascript
fastify.setErrorHandler((error, request, reply) => {
  request.log.error(error)

  // Custom error types
  if (error.code === 'USER_NOT_FOUND') {
    reply.code(404).send({ error: 'User not found' })
    return
  }

  // Validation errors
  if (error.validation) {
    reply.code(400).send({
      error: 'Validation failed',
      details: error.validation
    })
    return
  }

  // Default error
  reply.code(error.statusCode || 500).send({
    error: 'Internal Server Error'
  })
})

// Routes are clean
fastify.get('/users', async (request, reply) => {
  return getUsers()
})

fastify.get('/posts', async (request, reply) => {
  return getPosts()
})
```

### 4.3 Don't Send After Errors in Async

Once `reply.send()` is called, subsequent errors are silently ignored.

**Incorrect: error after send**

```javascript
fastify.get('/data', async (request, reply) => {
  reply.send({ data: 'value' })

  // This error is never reported!
  await doSomethingThatMightFail()
})
```

**Correct: return data**

```javascript
fastify.get('/data', async (request, reply) => {
  const data = { data: 'value' }
  await doSomethingThatMightFail() // Error is caught
  return data
})
```

### 4.4 Return Appropriate Status Codes

Use correct HTTP status codes for different error types.

**Incorrect: always 500**

```javascript
fastify.get('/user/:id', async (request, reply) => {
  const user = await getUserById(request.params.id)

  if (!user) {
    throw new Error('User not found') // Returns 500
  }

  return user
})
```

**Correct: use appropriate codes**

```javascript
fastify.get('/user/:id', async (request, reply) => {
  const user = await getUserById(request.params.id)

  if (!user) {
    reply.code(404).send({ error: 'User not found' })
    return
  }

  return user
})

// Or use custom error class
class NotFoundError extends Error {
  constructor(message) {
    super(message)
    this.statusCode = 404
  }
}

fastify.setErrorHandler((error, request, reply) => {
  reply.code(error.statusCode || 500).send({
    error: error.message
  })
})

fastify.get('/user/:id', async (request, reply) => {
  const user = await getUserById(request.params.id)

  if (!user) {
    throw new NotFoundError('User not found') // Returns 404
  }

  return user
})
```

### 4.5 Log Errors with Context

Include request context in error logs for debugging.

**Incorrect: minimal error logging**

```javascript
fastify.setErrorHandler((error, request, reply) => {
  console.log(error.message)
  reply.code(500).send({ error: 'Internal Server Error' })
})
```

**Correct: log with context**

```javascript
fastify.setErrorHandler((error, request, reply) => {
  request.log.error({
    err: error,
    req: {
      method: request.method,
      url: request.url,
      params: request.params,
      query: request.query,
      headers: request.headers,
      id: request.id
    }
  }, 'Request error')

  reply.code(error.statusCode || 500).send({
    error: 'Internal Server Error'
  })
})
```

---

## 5. Route Optimization

**Impact: MEDIUM-HIGH**

Fastify's routing is one of the fastest available, but improper usage can negate these benefits.

### 5.1 Use Radix Tree Routing Efficiently

Fastify uses a radix tree for fast route matching. Understanding how it works helps you write efficient routes.

**Incorrect: many similar routes**

```javascript
fastify.get('/api/users/active', async () => { /* ... */ })
fastify.get('/api/users/inactive', async () => { /* ... */ })
fastify.get('/api/users/banned', async () => { /* ... */ })
fastify.get('/api/users/pending', async () => { /* ... */ })
```

**Correct: use parameters**

```javascript
fastify.get('/api/users/:status', async (request, reply) => {
  const { status } = request.params

  const validStatuses = ['active', 'inactive', 'banned', 'pending']
  if (!validStatuses.includes(status)) {
    reply.code(400).send({ error: 'Invalid status' })
    return
  }

  return getUsersByStatus(status)
})

// Or use schema validation
fastify.get('/api/users/:status', {
  schema: {
    params: {
      type: 'object',
      properties: {
        status: {
          type: 'string',
          enum: ['active', 'inactive', 'banned', 'pending']
        }
      }
    }
  }
}, async (request, reply) => {
  return getUsersByStatus(request.params.status)
})
```

### 5.2 Avoid Dynamic Route Conflicts

Be careful with route parameter conflicts. Fastify matches routes in order of registration.

**Incorrect: conflicting routes**

```javascript
fastify.get('/users/:id', async (request, reply) => {
  return getUserById(request.params.id)
})

// This route is never reached!
fastify.get('/users/me', async (request, reply) => {
  return getCurrentUser(request)
})
```

**Correct: static routes first**

```javascript
// Register static routes before dynamic ones
fastify.get('/users/me', async (request, reply) => {
  return getCurrentUser(request)
})

fastify.get('/users/:id', async (request, reply) => {
  return getUserById(request.params.id)
})
```

### 5.3 Precompile Validators

Schemas are precompiled at registration time. Don't create validators at runtime.

**Incorrect: runtime validation**

```javascript
fastify.get('/search', async (request, reply) => {
  const ajv = new Ajv()
  const validate = ajv.compile({
    type: 'object',
    properties: {
      query: { type: 'string' }
    }
  })

  if (!validate(request.query)) {
    throw new Error('Invalid query')
  }

  return search(request.query.query)
})
```

**Correct: use route schema**

```javascript
fastify.get('/search', {
  schema: {
    querystring: {
      type: 'object',
      required: ['query'],
      properties: {
        query: { type: 'string', minLength: 1 }
      }
    }
  }
}, async (request, reply) => {
  // Validation already done
  return search(request.query.query)
})
```

### 5.4 Use Decorators for Shared Utils

Decorators provide optimized property access compared to importing modules.

**Incorrect: importing in every route**

```javascript
const { formatDate } = require('../utils')

fastify.get('/users', async (request, reply) => {
  const users = await getUsers()
  return users.map(u => ({
    ...u,
    createdAt: formatDate(u.createdAt)
  }))
})
```

**Correct: use decorator for frequently used utils**

```javascript
const { formatDate } = require('../utils')

fastify.decorate('formatDate', formatDate)

fastify.get('/users', async (request, reply) => {
  const users = await getUsers()
  return users.map(u => ({
    ...u,
    createdAt: fastify.formatDate(u.createdAt)
  }))
})
```

Note: Only use decorators for frequently used utilities. For one-off usage, direct imports are fine.

### 5.5 Minimize Hook Overhead

Hooks add overhead to every matching request. Use them judiciously.

**Incorrect: hooks for single route**

```javascript
fastify.addHook('preHandler', async (request, reply) => {
  if (request.url === '/admin') {
    await checkAdminAuth(request)
  }
})

fastify.get('/admin', async (request, reply) => {
  return getAdminData()
})

fastify.get('/users', async (request, reply) => {
  return getUsers() // Hook runs but does nothing
})
```

**Correct: use route-level hooks**

```javascript
fastify.get('/admin', {
  preHandler: async (request, reply) => {
    await checkAdminAuth(request)
  }
}, async (request, reply) => {
  return getAdminData()
})

fastify.get('/users', async (request, reply) => {
  return getUsers() // No hook overhead
})
```

**Or use encapsulation:**

```javascript
fastify.register(async (fastify) => {
  // Hook only runs for routes in this context
  fastify.addHook('preHandler', async (request, reply) => {
    await checkAdminAuth(request)
  })

  fastify.get('/admin', async (request, reply) => {
    return getAdminData()
  })

  fastify.get('/admin/users', async (request, reply) => {
    return getAdminUsers()
  })
}, { prefix: '/admin' })

// Hook doesn't run here
fastify.get('/users', async (request, reply) => {
  return getUsers()
})
```

---

## 6. Logging & Monitoring

**Impact: MEDIUM**

Logging is essential for debugging and monitoring, but improper logging can significantly impact performance.

### 6.1 Use Pino Logger

Fastify uses Pino, the fastest Node.js logger. Use the built-in logger instead of console.log or other loggers.

**Incorrect: console.log**

```javascript
const fastify = require('fastify')()

fastify.get('/users', async (request, reply) => {
  console.log('Fetching users') // Slow, blocking
  const users = await getUsers()
  console.log(`Found ${users.length} users`)
  return users
})
```

**Correct: use Pino logger**

```javascript
const fastify = require('fastify')({
  logger: true
})

fastify.get('/users', async (request, reply) => {
  request.log.info('Fetching users') // Fast, async
  const users = await getUsers()
  request.log.info({ count: users.length }, 'Found users')
  return users
})
```

**Configure Pino for production:**

```javascript
const fastify = require('fastify')({
  logger: {
    level: process.env.LOG_LEVEL || 'info',
    serializers: {
      req(request) {
        return {
          method: request.method,
          url: request.url,
          headers: request.headers,
          hostname: request.hostname,
          remoteAddress: request.ip,
          remotePort: request.socket.remotePort
        }
      },
      res(reply) {
        return {
          statusCode: reply.statusCode
        }
      }
    }
  }
})
```

### 6.2 Avoid Synchronous Logging

Never use synchronous logging methods in production. They block the event loop.

**Incorrect: synchronous logging**

```javascript
const pino = require('pino')
const logger = pino({
  sync: true // Blocks event loop!
})

const fastify = require('fastify')({
  logger: logger
})
```

**Correct: async logging (default)**

```javascript
const fastify = require('fastify')({
  logger: true // Async by default
})
```

### 6.3 Use Appropriate Log Levels

Use the right log level for the right message. Avoid debug logs in production.

**Incorrect: everything is info**

```javascript
fastify.get('/users', async (request, reply) => {
  request.log.info('Starting to fetch users')
  request.log.info('Connecting to database')
  request.log.info('Executing query')
  const users = await getUsers()
  request.log.info('Query completed')
  request.log.info('Formatting results')
  request.log.info('Returning response')
  return users
})
```

**Correct: use appropriate levels**

```javascript
fastify.get('/users', async (request, reply) => {
  request.log.debug('Fetching users from database')

  try {
    const users = await getUsers()
    request.log.debug({ count: users.length }, 'Users fetched successfully')
    return users
  } catch (error) {
    request.log.error({ err: error }, 'Failed to fetch users')
    throw error
  }
})
```

**Log levels:**
- `fatal` - Application is shutting down
- `error` - Error that needs attention
- `warn` - Warning that should be reviewed
- `info` - Important application events
- `debug` - Detailed information for debugging
- `trace` - Very detailed information

### 6.4 Include Request IDs

Use request IDs to trace requests across services.

**Incorrect: no request ID**

```javascript
const fastify = require('fastify')({
  logger: true
})

fastify.get('/users', async (request, reply) => {
  request.log.info('Fetching users')
  return getUsers()
})
```

**Correct: enable request IDs**

```javascript
const fastify = require('fastify')({
  logger: true,
  requestIdHeader: 'x-request-id',
  requestIdLogLabel: 'reqId',
  genReqId: (req) => {
    return req.headers['x-request-id'] || generateUUID()
  }
})

fastify.get('/users', async (request, reply) => {
  // Logs automatically include reqId
  request.log.info('Fetching users')
  return getUsers()
})
```

### 6.5 Use Child Loggers for Context

Create child loggers to add context to all subsequent logs.

**Incorrect: repeating context**

```javascript
fastify.get('/user/:id', async (request, reply) => {
  request.log.info({ userId: request.params.id }, 'Fetching user')

  const user = await getUser(request.params.id)
  request.log.info({ userId: request.params.id }, 'User found')

  await updateUserActivity(request.params.id)
  request.log.info({ userId: request.params.id }, 'Activity updated')

  return user
})
```

**Correct: use child logger**

```javascript
fastify.get('/user/:id', async (request, reply) => {
  const logger = request.log.child({ userId: request.params.id })

  logger.info('Fetching user')

  const user = await getUser(request.params.id)
  logger.info('User found')

  await updateUserActivity(request.params.id)
  logger.info('Activity updated')

  return user
})
```

---

## 7. Deployment & Infrastructure

**Impact: MEDIUM**

Proper deployment configuration ensures Fastify runs optimally in production.

### 7.1 Use Reverse Proxy

Never expose Fastify directly to the internet. Use a reverse proxy like Nginx or HAProxy.

**Incorrect: direct internet exposure**

```javascript
const fastify = require('fastify')({
  https: {
    key: fs.readFileSync('server.key'),
    cert: fs.readFileSync('server.cert')
  }
})

fastify.listen({ port: 443, host: '0.0.0.0' })
```

**Correct: behind reverse proxy**

```javascript
const fastify = require('fastify')({
  trustProxy: true // Trust X-Forwarded-* headers
})

// Listen on localhost, let proxy handle HTTPS
fastify.listen({ port: 3000, host: '127.0.0.1' })
```

**Nginx configuration:**

```nginx
server {
  listen 80;
  listen 443 ssl;
  server_name example.com;

  ssl_certificate /path/to/cert.pem;
  ssl_certificate_key /path/to/key.pem;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_cache_bypass $http_upgrade;
  }
}
```

### 7.2 Optimize vCPU Allocation

Choose vCPU allocation based on your optimization goal.

**For low latency:**

```yaml
# Kubernetes deployment
resources:
  requests:
    cpu: 2000m
  limits:
    cpu: 2000m
```

2 vCPU allows garbage collection and libuv threadpool to operate efficiently without blocking.

**For high throughput:**

```yaml
# Kubernetes deployment
resources:
  requests:
    cpu: 1000m  # or even 100m-200m
  limits:
    cpu: 1000m
```

Run more instances with fewer vCPUs each for better horizontal scaling.

### 7.3 Listen on 0.0.0.0 in Containers

Fastify defaults to `127.0.0.1`, which is unreachable from Kubernetes probes.

**Incorrect: default host**

```javascript
fastify.listen({ port: 3000 })
// Defaults to 127.0.0.1 - unreachable in Kubernetes
```

**Correct: listen on all interfaces**

```javascript
fastify.listen({
  port: 3000,
  host: '0.0.0.0' // Reachable from Kubernetes probes
})
```

### 7.4 Configure Readiness Probes

Ensure Kubernetes can properly check if your app is ready.

**Incorrect: no readiness endpoint**

```javascript
const fastify = require('fastify')()

fastify.get('/', async () => ({ hello: 'world' }))

fastify.listen({ port: 3000, host: '0.0.0.0' })
```

**Correct: add health check endpoints**

```javascript
const fastify = require('fastify')()

// Liveness probe - is the app running?
fastify.get('/health', async () => ({ status: 'ok' }))

// Readiness probe - is the app ready to serve traffic?
fastify.get('/ready', async (request, reply) => {
  try {
    // Check database connection
    await fastify.db.query('SELECT 1')

    return { status: 'ready' }
  } catch (error) {
    reply.code(503).send({ status: 'not ready' })
  }
})

fastify.get('/', async () => ({ hello: 'world' }))

fastify.listen({ port: 3000, host: '0.0.0.0' })
```

**Kubernetes configuration:**

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 7.5 Use Performance Testing Tools

Use k6 or autocannon to test and optimize your application.

**Using autocannon:**

```bash
npm install -g autocannon

# Basic test
autocannon http://localhost:3000

# Custom test
autocannon -c 100 -d 30 -p 10 http://localhost:3000/users
# -c 100: 100 concurrent connections
# -d 30: run for 30 seconds
# -p 10: 10 pipelining factor
```

**Using k6:**

```javascript
// load-test.js
import http from 'k6/http'
import { check } from 'k6'

export let options = {
  stages: [
    { duration: '30s', target: 50 },
    { duration: '1m', target: 100 },
    { duration: '30s', target: 0 }
  ]
}

export default function() {
  let res = http.get('http://localhost:3000/users')
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 100ms': (r) => r.timings.duration < 100
  })
}
```

```bash
k6 run load-test.js
```

---

## References

1. [https://fastify.dev](https://fastify.dev)
2. [https://fastify.dev/docs/latest/Guides/Recommendations/](https://fastify.dev/docs/latest/Guides/Recommendations/)
3. [https://fastify.dev/docs/latest/Reference/Hooks/](https://fastify.dev/docs/latest/Reference/Hooks/)
4. [https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/)
5. [https://fastify.dev/docs/latest/Reference/Plugins/](https://fastify.dev/docs/latest/Reference/Plugins/)
6. [https://github.com/fastify/fastify](https://github.com/fastify/fastify)
7. [https://github.com/pinojs/pino](https://github.com/pinojs/pino)
8. [https://github.com/fastify/fastify-plugin](https://github.com/fastify/fastify-plugin)
