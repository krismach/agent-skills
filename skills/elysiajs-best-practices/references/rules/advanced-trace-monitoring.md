---
title: Use Trace for Performance Monitoring
impact: LOW
impactDescription: Identifies performance bottlenecks
tags: advanced, trace, monitoring, debugging
---

## Use Trace for Performance Monitoring

**Impact: LOW (Identifies performance bottlenecks)**

Use Elysia's trace feature to inject timing code before and after each lifecycle event, helping identify performance bottlenecks in production or development.

**Pattern: Basic performance tracing:**

```typescript
app
  .trace(async ({ onHandle, set }) => {
    const { time, end } = await onHandle

    console.log('Handler started:', time)
    const result = await end
    console.log('Handler completed:', result.time - time, 'ms')
  })

  .get('/slow', async () => {
    await Bun.sleep(1000)
    return 'Done'
  })
// Logs: Handler started: 1234567890
// Logs: Handler completed: 1000 ms
```

**Advanced: Trace all lifecycle events:**

```typescript
app
  .trace(async ({ request, onBeforeHandle, onHandle, onAfterHandle }) => {
    const requestStart = performance.now()

    console.log(`[${request.method}] ${new URL(request.url).pathname}`)

    await onBeforeHandle
    const beforeHandleTime = performance.now() - requestStart
    console.log(`  beforeHandle: ${beforeHandleTime.toFixed(2)}ms`)

    await onHandle
    const handleTime = performance.now() - requestStart - beforeHandleTime
    console.log(`  handle: ${handleTime.toFixed(2)}ms`)

    await onAfterHandle
    const totalTime = performance.now() - requestStart
    console.log(`  total: ${totalTime.toFixed(2)}ms`)
  })

  .get('/api/users', () => getUsers())
```

**Pattern: Structured logging with trace:**

```typescript
const logger = {
  info: (data: any) => console.log(JSON.stringify(data)),
  error: (data: any) => console.error(JSON.stringify(data))
}

app
  .derive(({ request }) => ({
    requestId: crypto.randomUUID(),
    timestamp: Date.now()
  }))

  .trace(async ({ requestId, request, onHandle, set }) => {
    const start = performance.now()
    const { time: handleStart, end: handleEnd } = await onHandle

    const result = await handleEnd
    const duration = performance.now() - start

    logger.info({
      requestId,
      method: request.method,
      path: new URL(request.url).pathname,
      status: set.status,
      duration: `${duration.toFixed(2)}ms`,
      handleDuration: `${(result.time - handleStart).toFixed(2)}ms`
    })
  })

  .get('/api/data', () => fetchData())
```

**Conditional tracing (development only):**

```typescript
const app = new Elysia()

if (Bun.env.NODE_ENV === 'development') {
  app.trace(async ({ onHandle, path }) => {
    const { time, end } = await onHandle
    const result = await end

    if (result.time - time > 100) {
      console.warn(`⚠️  Slow handler: ${path} took ${result.time - time}ms`)
    }
  })
}

app.get('/data', () => getData())
```

**Pattern: APM integration:**

```typescript
import * as Sentry from '@sentry/node'

app.trace(async ({
  request,
  onBeforeHandle,
  onHandle,
  onAfterHandle,
  set
}) => {
  const transaction = Sentry.startTransaction({
    op: 'http.server',
    name: `${request.method} ${new URL(request.url).pathname}`
  })

  try {
    const beforeSpan = transaction.startChild({ op: 'beforeHandle' })
    await onBeforeHandle
    beforeSpan.finish()

    const handleSpan = transaction.startChild({ op: 'handle' })
    await onHandle
    handleSpan.finish()

    const afterSpan = transaction.startChild({ op: 'afterHandle' })
    await onAfterHandle
    afterSpan.finish()

    transaction.setHttpStatus(set.status)
    transaction.finish()
  } catch (error) {
    transaction.finish()
    Sentry.captureException(error)
    throw error
  }
})
```

**Trace specific lifecycle events:**

```typescript
app
  .trace(async ({ onBeforeHandle }) => {
    const { name, begin, end } = await onBeforeHandle

    await begin
    console.log(`${name} started`)

    const result = await end
    console.log(`${name} completed in ${result.time}ms`)
  })

  .onBeforeHandle(async () => {
    // This will be traced
    await validateAuth()
  })
```

**Performance budgets with trace:**

```typescript
const PERFORMANCE_BUDGETS = {
  '/api/fast': 50,      // 50ms budget
  '/api/normal': 200,   // 200ms budget
  '/api/slow': 1000     // 1s budget
}

app.trace(async ({ path, onHandle }) => {
  const { time: start, end } = await onHandle
  const result = await end

  const duration = result.time - start
  const budget = PERFORMANCE_BUDGETS[path]

  if (budget && duration > budget) {
    logger.warn({
      message: 'Performance budget exceeded',
      path,
      duration: `${duration}ms`,
      budget: `${budget}ms`,
      exceeded: `${duration - budget}ms`
    })
  }
})
```

**Trace with metrics collection:**

```typescript
const metrics = {
  requests: 0,
  totalDuration: 0,
  errors: 0
}

app
  .trace(async ({ onHandle, onError }) => {
    metrics.requests++

    const { time: start, end } = await onHandle
    const result = await end

    metrics.totalDuration += (result.time - start)

    // Track errors
    onError(() => {
      metrics.errors++
    })
  })

  .get('/metrics', () => ({
    requests: metrics.requests,
    avgDuration: metrics.totalDuration / metrics.requests,
    errorRate: (metrics.errors / metrics.requests) * 100
  }))
```

Reference: [ElysiaJS Trace Documentation](https://elysiajs.com/patterns/trace)
