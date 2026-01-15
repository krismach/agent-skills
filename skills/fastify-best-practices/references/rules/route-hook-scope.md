# Minimize Hook Overhead

## Section

Route Optimization

## Summary

Use route-level hooks or encapsulation instead of global hooks to minimize overhead.

## Incorrect

```javascript
// Global hook runs for all routes
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

## Correct

```javascript
// Route-level hook
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

## Why

Global hooks add overhead to every request. Route-level hooks or encapsulation ensure hooks only run where needed.
