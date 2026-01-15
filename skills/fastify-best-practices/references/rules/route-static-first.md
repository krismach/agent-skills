# Register Static Routes Before Dynamic

## Section

Route Optimization

## Summary

Static routes must be registered before dynamic routes to avoid matching conflicts.

## Incorrect

```javascript
fastify.get('/users/:id', async (request, reply) => {
  return getUserById(request.params.id)
})

// This route is never reached!
fastify.get('/users/me', async (request, reply) => {
  return getCurrentUser(request)
})
```

## Correct

```javascript
// Register static routes before dynamic ones
fastify.get('/users/me', async (request, reply) => {
  return getCurrentUser(request)
})

fastify.get('/users/:id', async (request, reply) => {
  return getUserById(request.params.id)
})
```

## Why

Fastify's radix tree router matches routes in order of registration. Dynamic routes will match static paths if registered first.
