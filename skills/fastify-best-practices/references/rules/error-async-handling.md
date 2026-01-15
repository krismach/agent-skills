# Let Fastify Handle Async Errors

## Section

Error Handling

## Summary

Fastify automatically catches errors in async handlers. Don't wrap everything in try-catch.

## Incorrect

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

## Correct

```javascript
fastify.get('/users', async (request, reply) => {
  const users = await getUsers()
  return users
  // Errors are automatically caught and reported
})
```

## Why

Fastify's automatic error handling is one of its key advantages over Express. Unnecessary try-catch adds boilerplate and can mask errors.
