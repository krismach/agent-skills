# Avoid Mixing Callbacks and Promises

## Section

Async & Request Lifecycle

## Summary

Never mix callback style (using `done`) with promises/async-await. The hook chain will execute twice.

## Incorrect

```javascript
fastify.addHook('onRequest', async (request, reply, done) => {
  await doSomethingAsync()
  done() // Hook executes twice!
})
```

## Correct

```javascript
fastify.addHook('onRequest', async (request, reply) => {
  await doSomethingAsync()
  // No done() needed
})
```

## Why

Mixing callbacks and async/promise causes the hook chain to execute twice, leading to unpredictable behavior and potential race conditions.
