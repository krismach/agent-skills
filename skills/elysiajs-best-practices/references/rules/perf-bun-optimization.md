---
title: Leverage Bun's Native APIs
impact: MEDIUM
impactDescription: Faster than Node.js equivalents
tags: performance, bun, apis, optimization
---

## Leverage Bun's Native APIs

**Impact: MEDIUM (Faster than Node.js equivalents)**

Bun provides optimized native APIs that are significantly faster than Node.js equivalents. Use them for file operations, hashing, environment variables, and more.

**Incorrect (using Node.js APIs):**

```typescript
import fs from 'fs/promises'
import crypto from 'crypto'

app.get('/file', async () => {
  const content = await fs.readFile('./data.json', 'utf-8')
  return JSON.parse(content)
})

app.post('/hash', async ({ body }) => {
  const hash = crypto.createHash('sha256')
    .update(body.data)
    .digest('hex')
  return { hash }
})
```

**Correct (using Bun native APIs):**

```typescript
app.get('/file', async () => {
  const file = Bun.file('./data.json')
  return await file.json() // Faster native parsing
})

app.post('/hash', async ({ body }) => {
  const hasher = new Bun.CryptoHasher('sha256')
  hasher.update(body.data)
  return { hash: hasher.digest('hex') }
})
```

**Pattern: Efficient file operations:**

```typescript
app
  // Read file
  .get('/download/:filename', async ({ params }) => {
    const file = Bun.file(`./uploads/${params.filename}`)

    if (!await file.exists()) {
      throw new Error('File not found')
    }

    return file // Bun automatically streams
  })

  // Write file
  .post('/upload', async ({ body }) => {
    await Bun.write('./uploads/data.json', JSON.stringify(body))
    return { success: true }
  })

  // Check file exists
  .get('/check/:filename', async ({ params }) => {
    const file = Bun.file(`./uploads/${params.filename}`)
    return { exists: await file.exists() }
  })
```

**Bun.sleep for delays:**

```typescript
// Incorrect: using setTimeout with Promise
const sleep = (ms: number) => new Promise(resolve => setTimeout(resolve, ms))

app.get('/delayed', async () => {
  await sleep(1000)
  return { data: 'delayed' }
})

// Correct: using Bun.sleep (more efficient)
app.get('/delayed', async () => {
  await Bun.sleep(1000)
  return { data: 'delayed' }
})
```

**Fast hashing and encoding:**

```typescript
app
  // Fast password hashing
  .post('/hash-password', async ({ body }) => {
    const hash = await Bun.password.hash(body.password, {
      algorithm: 'argon2id', // or 'bcrypt'
      memoryCost: 19456,
      timeCost: 2
    })
    return { hash }
  })

  // Verify password
  .post('/verify-password', async ({ body }) => {
    const isValid = await Bun.password.verify(
      body.password,
      body.hash
    )
    return { valid: isValid }
  })

  // Fast crypto
  .get('/random', () => {
    const bytes = new Uint8Array(32)
    crypto.getRandomValues(bytes) // Fast in Bun
    return { random: Buffer.from(bytes).toString('hex') }
  })
```

**Environment variables:**

```typescript
// Bun.env is faster than process.env
const config = {
  port: Bun.env.PORT ?? '3000',
  dbUrl: Bun.env.DATABASE_URL,
  nodeEnv: Bun.env.NODE_ENV ?? 'development'
}

app
  .get('/config', () => ({
    environment: Bun.env.NODE_ENV,
    version: Bun.env.APP_VERSION
  }))
```

**Bun.gc for memory management:**

```typescript
app.post('/cleanup', async ({ body }) => {
  // Process large batch
  const results = await processBatch(body.items)

  // Force garbage collection after heavy work
  if (Bun.gc) {
    Bun.gc(true)
  }

  return results
})
```

**Pattern: Optimized JSON handling:**

```typescript
app
  // Fast JSON parsing from file
  .get('/data', async () => {
    const file = Bun.file('./large-data.json')
    return await file.json() // Faster than JSON.parse(await file.text())
  })

  // Fast JSON writing
  .post('/save', async ({ body }) => {
    await Bun.write(
      './output.json',
      JSON.stringify(body, null, 2)
    )
    return { saved: true }
  })
```

**Using Bun.serve features (advanced):**

```typescript
// While Elysia abstracts this, you can access low-level features
const server = app.listen(3000, (server) => {
  console.log(`Server started on ${server.hostname}:${server.port}`)

  // Access underlying Bun server
  console.log('Pending requests:', server.pendingRequests)
  console.log('Pending websockets:', server.pendingWebSockets)
})

// Graceful shutdown
process.on('SIGTERM', () => {
  server.stop()
})
```

**File watching (development):**

```typescript
if (Bun.env.NODE_ENV === 'development') {
  const watcher = Bun.file('./config.json').watch()

  for await (const event of watcher) {
    console.log('Config changed, reloading...')
    // Reload config
  }
}
```

**Subprocess with Bun:**

```typescript
app.post('/process', async ({ body }) => {
  const proc = Bun.spawn(['python', 'process.py', body.input], {
    stdout: 'pipe'
  })

  const output = await new Response(proc.stdout).text()
  await proc.exited

  return { output }
})
```

Reference: [Bun Runtime APIs](https://bun.sh/docs/api)
