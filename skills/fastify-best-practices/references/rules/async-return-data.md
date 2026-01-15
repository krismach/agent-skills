# Return Data Instead of reply.send()

## Section

Async & Request Lifecycle

## Summary

When using async/await, return the data from the handler instead of calling `reply.send()` to prevent silent errors.

## Incorrect

```javascript
fastify.get('/user/:id', async (request, reply) => {
  const user = await getUserById(request.params.id)
  reply.send(user) // Works, but not recommended
})
```

## Correct

```javascript
fastify.get('/user/:id', async (request, reply) => {
  const user = await getUserById(request.params.id)
  return user // Fastify handles serialization and errors
})
```

## Why

If an error occurs after `reply.send()`, it won't be caught or reported. Returning data allows Fastify to automatically serialize the return value and catch any errors that occur.
