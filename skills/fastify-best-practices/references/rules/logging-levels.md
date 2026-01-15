# Use Appropriate Log Levels

## Section

Logging & Monitoring

## Summary

Use the right log level for the right message to avoid noise in production.

## Log Levels

- `fatal` - Application is shutting down
- `error` - Error that needs attention
- `warn` - Warning that should be reviewed
- `info` - Important application events
- `debug` - Detailed information for debugging
- `trace` - Very detailed information

## Incorrect

```javascript
// Everything is info
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

## Correct

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

## Why

Proper log levels make it easier to filter logs in production and reduce log volume.
