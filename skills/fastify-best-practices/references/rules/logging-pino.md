# Use Pino Logger

## Section

Logging & Monitoring

## Summary

Use Fastify's built-in Pino logger instead of console.log or other loggers.

## Incorrect

```javascript
const fastify = require('fastify')()

fastify.get('/users', async (request, reply) => {
  console.log('Fetching users') // Slow, blocking
  const users = await getUsers()
  console.log(`Found ${users.length} users`)
  return users
})
```

## Correct

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

## Why

Pino is the fastest Node.js logger and integrates seamlessly with Fastify. console.log is synchronous and blocks the event loop.
