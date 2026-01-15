# Keep Plugins Focused

## Section

Plugin Architecture

## Summary

Each plugin should have a single, clear purpose for better testability and reusability.

## Incorrect

```javascript
async function megaPlugin(fastify, options) {
  // Database connection
  const db = await connectToDatabase()
  fastify.decorate('db', db)

  // Authentication
  fastify.decorate('authenticate', async (request) => { /* ... */ })

  // Email sending
  fastify.decorate('sendEmail', async (to, subject, body) => { /* ... */ })

  // Routes
  fastify.get('/users', async () => { /* ... */ })
  fastify.post('/login', async () => { /* ... */ })
}
```

## Correct

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
```

## Why

Focused plugins are easier to test, maintain, and reuse across projects. They also make it easier to understand what each plugin does.
