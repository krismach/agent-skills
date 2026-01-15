# Use fastify-plugin for Shared Decorators

## Section

Plugin Architecture

## Summary

Use `fastify-plugin` when you need decorators or hooks to be available in parent scope.

## Incorrect

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

## Correct

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

## Why

By default, plugins create an encapsulated context. Use fastify-plugin to break encapsulation when you need to share decorators or hooks with parent scopes.
