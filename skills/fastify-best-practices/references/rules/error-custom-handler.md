# Use setErrorHandler for Custom Errors

## Section

Error Handling

## Summary

Define a global error handler to customize error responses instead of handling errors in every route.

## Incorrect

```javascript
// Handling errors in every route
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

## Correct

```javascript
// Global error handler
fastify.setErrorHandler((error, request, reply) => {
  request.log.error(error)

  if (error.code === 'USER_NOT_FOUND') {
    reply.code(404).send({ error: 'User not found' })
    return
  }

  if (error.validation) {
    reply.code(400).send({
      error: 'Validation failed',
      details: error.validation
    })
    return
  }

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

## Why

A global error handler reduces code duplication and ensures consistent error responses across your application.
