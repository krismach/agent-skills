# Use Lifecycle Hooks Efficiently

## Section

Async & Request Lifecycle

## Summary

Choose the right hook for the right task based on when body parsing and validation occur.

## Hook Order

1. `onRequest` - Before parsing (no body available)
2. `preParsing` - Before body parsing
3. `preValidation` - After parsing, before validation
4. `preHandler` - After validation, before handler
5. Handler execution
6. `preSerialization` - Before serialization
7. `onSend` - Before sending response
8. `onResponse` - After response sent

## Incorrect

```javascript
// Don't parse body if you don't need it
fastify.addHook('preHandler', async (request, reply) => {
  // Body is already parsed, wasting resources
  if (!request.headers.authorization) {
    throw new Error('Unauthorized')
  }
})
```

## Correct

```javascript
// Check auth before parsing body
fastify.addHook('onRequest', async (request, reply) => {
  if (!request.headers.authorization) {
    throw new Error('Unauthorized')
  }
})
```

## Why

Using the earliest possible hook prevents unnecessary work like body parsing when a request will be rejected anyway.
