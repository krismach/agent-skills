# Share Schemas with addSchema()

## Section

Schema Validation & Serialization

## Summary

Reuse common schemas across routes using the `addSchema()` API for better maintainability.

## Incorrect

```javascript
// Duplicating schemas across routes
fastify.get('/user/:id', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          id: { type: 'number' },
          name: { type: 'string' },
          email: { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => { /* ... */ })

fastify.post('/user', {
  schema: {
    response: {
      201: {
        type: 'object',
        properties: {
          id: { type: 'number' },
          name: { type: 'string' },
          email: { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => { /* ... */ })
```

## Correct

```javascript
// Register shared schema once
fastify.addSchema({
  $id: 'user',
  type: 'object',
  properties: {
    id: { type: 'number' },
    name: { type: 'string' },
    email: { type: 'string' }
  }
})

// Reference it in routes
fastify.get('/user/:id', {
  schema: {
    response: {
      200: { $ref: 'user#' }
    }
  }
}, async (request, reply) => { /* ... */ })

fastify.post('/user', {
  schema: {
    response: {
      201: { $ref: 'user#' }
    }
  }
}, async (request, reply) => { /* ... */ })
```

## Why

Shared schemas improve maintainability and enable schema composition while ensuring consistency across your API.
