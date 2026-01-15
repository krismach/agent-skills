# Define JSON Schema for All Routes

## Section

Schema Validation & Serialization

## Summary

Always define schemas for route validation. Fastify compiles these into highly performant functions using AJV v8.

## Incorrect

```javascript
fastify.post('/user', async (request, reply) => {
  // Manual validation is slower and error-prone
  if (!request.body.email || typeof request.body.email !== 'string') {
    throw new Error('Invalid email')
  }
  if (!request.body.age || typeof request.body.age !== 'number') {
    throw new Error('Invalid age')
  }

  return createUser(request.body)
})
```

## Correct

```javascript
fastify.post('/user', {
  schema: {
    body: {
      type: 'object',
      required: ['email', 'age'],
      properties: {
        email: { type: 'string', format: 'email' },
        age: { type: 'number', minimum: 0, maximum: 120 }
      }
    }
  }
}, async (request, reply) => {
  // request.body is validated - no manual checks needed
  return createUser(request.body)
})
```

## Why

Schema validation is 10-100x faster than manual validation, provides automatic 400 responses on validation failure, and self-documents your API.
