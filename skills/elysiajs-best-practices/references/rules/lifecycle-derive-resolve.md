---
title: Use derive for Pre-Validation, resolve for Post-Validation
impact: HIGH
impactDescription: Proper data flow and type safety
tags: lifecycle, derive, resolve, validation
---

## Use derive for Pre-Validation, resolve for Post-Validation

**Impact: HIGH (Proper data flow and type safety)**

Use `derive` to add properties before validation (working with raw data) and `resolve` to add properties after validation (working with validated data). This ensures correct data flow and type safety.

**Incorrect (using resolve for pre-validation logic):**

```typescript
app
  .resolve(({ headers }) => ({
    // headers are not validated yet, could be malformed
    userId: extractUserId(headers.authorization)
  }))
  .get('/user', ({ userId }) => getUserData(userId))
```

**Correct (derive for pre-validation, resolve for post-validation):**

```typescript
app
  // derive: Extract user ID from raw headers (before validation)
  .derive(({ headers }) => ({
    userId: extractUserId(headers.authorization)
  }))

  // Now validate the extracted userId
  .get('/user/:id', ({ userId, params }) => {
    // userId is available, params.id is validated
    return getUserData(params.id)
  }, {
    params: t.Object({
      id: t.Numeric()
    })
  })

  // resolve: Add properties that depend on validated data
  .resolve(({ userId }) => ({
    user: userId ? getUserFromCache(userId) : null
  }))

  .get('/profile', ({ user }) => {
    // user is based on validated userId
    if (!user) throw new Error('Not authenticated')
    return user
  })
```

**Pattern: Request context enrichment:**

```typescript
app
  // Step 1: derive - Extract from raw request
  .derive(({ request, headers }) => ({
    requestId: headers['x-request-id'] ?? crypto.randomUUID(),
    ip: request.headers.get('x-forwarded-for') ?? 'unknown',
    userAgent: headers['user-agent']
  }))

  // Step 2: Schema validation happens here
  .get('/data', ({ body, requestId }) => {
    log.info({ requestId }, 'Processing request')
    return processData(body)
  }, {
    body: t.Object({
      data: t.String()
    })
  })

  // Step 3: resolve - Add properties based on validated data
  .resolve(async ({ body, requestId }) => {
    // body is now validated
    const cached = await getFromCache(body.data)
    return { cached, requestId }
  })
```

**Lifecycle execution order:**

```typescript
// Request flow:
// 1. onRequest
// 2. onTransform
// 3. derive          ← Work with raw data
// 4. Schema validation ← Elysia validates here
// 5. resolve         ← Work with validated data
// 6. onBeforeHandle
// 7. Route handler
// 8. onAfterHandle
// 9. Response

new Elysia()
  .derive(() => {
    console.log('Before validation - raw data')
    return { step: 'derive' }
  })
  .resolve(() => {
    console.log('After validation - safe data')
    return { step: 'resolve' }
  })
```

**Best practice:**

```typescript
// Use derive for:
// - Extracting authentication tokens
// - Adding request metadata (ID, timestamp, IP)
// - Parsing custom headers
// - Any logic that needs raw request data

// Use resolve for:
// - Loading user data after auth validation
// - Fetching resources based on validated IDs
// - Computing derived values from validated input
// - Database queries using validated parameters
```

Reference: [ElysiaJS Lifecycle Hooks](https://elysiajs.com/essential/life-cycle#derive)
