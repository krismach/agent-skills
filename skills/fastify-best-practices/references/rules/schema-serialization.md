# Use Serialization Schemas

## Section

Schema Validation & Serialization

## Summary

Define response schemas for 100-400% performance boost using fast-json-stringify.

## Incorrect

```javascript
fastify.get('/users', async (request, reply) => {
  const users = await getUsers()
  return users // Uses slow JSON.stringify()
})
```

## Correct

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

## Why

fast-json-stringify is 100-400% faster than JSON.stringify() and automatically filters unwanted properties from responses.
