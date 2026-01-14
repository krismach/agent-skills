---
title: Use Streaming for Large Responses
impact: MEDIUM-HIGH
impactDescription: Reduces memory usage and improves TTFB
tags: response, streaming, performance, memory
---

## Use Streaming for Large Responses

**Impact: MEDIUM-HIGH (Reduces memory usage and improves TTFB)**

Use generator functions to stream large responses, reducing memory consumption and improving Time to First Byte. ElysiaJS automatically handles streaming when you return a generator.

**Incorrect (loading all data in memory):**

```typescript
app.get('/export', async () => {
  // Loads all records in memory - 100MB+ for large datasets
  const allRecords = await db.records.findMany()

  const csv = [
    'id,name,email,created',
    ...allRecords.map(r => `${r.id},${r.name},${r.email},${r.created}`)
  ].join('\n')

  return new Response(csv, {
    headers: { 'Content-Type': 'text/csv' }
  })
})
```

**Correct (streaming response):**

```typescript
app.get('/export', async function* () {
  // Stream header
  yield 'id,name,email,created\n'

  // Stream records in batches
  let cursor = 0
  const batchSize = 1000

  while (true) {
    const records = await db.records.findMany({
      take: batchSize,
      skip: cursor
    })

    if (records.length === 0) break

    for (const record of records) {
      yield `${record.id},${record.name},${record.email},${record.created}\n`
    }

    cursor += batchSize
  }
})
```

**Pattern: Server-Sent Events (SSE):**

```typescript
app.get('/events', async function* () {
  // SSE format: data: {json}\n\n
  let count = 0

  while (count < 10) {
    yield `data: ${JSON.stringify({ count, time: Date.now() })}\n\n`
    count++
    await Bun.sleep(1000) // Wait 1 second
  }
})

// Client usage:
// const events = new EventSource('/events')
// events.onmessage = (e) => console.log(JSON.parse(e.data))
```

**Streaming with cleanup:**

```typescript
app.get('/stream', async function* ({ request }) {
  const connection = await createConnection()

  try {
    for await (const chunk of connection.stream()) {
      // Check if client disconnected
      if (request.signal.aborted) {
        console.log('Client disconnected')
        break
      }

      yield chunk
    }
  } finally {
    // Cleanup is called even if client disconnects
    await connection.close()
  }
})
```

**Real-time data streaming:**

```typescript
app.get('/logs', async function* ({ query }) {
  const logStream = await getLogStream(query.service)

  yield `data: ${JSON.stringify({ status: 'connected' })}\n\n`

  for await (const log of logStream) {
    yield `data: ${JSON.stringify(log)}\n\n`
  }
}, {
  query: t.Object({
    service: t.String()
  })
})
```

**Pattern: Streaming JSON arrays:**

```typescript
app.get('/users/stream', async function* () {
  yield '['

  const users = await db.users.findMany()

  for (let i = 0; i < users.length; i++) {
    yield JSON.stringify(users[i])
    if (i < users.length - 1) yield ','
  }

  yield ']'
})
```

**Batch processing with progress updates:**

```typescript
app.post('/process', async function* ({ body }) {
  const items = body.items
  let processed = 0

  for (const item of items) {
    await processItem(item)
    processed++

    // Send progress update
    yield `data: ${JSON.stringify({
      progress: (processed / items.length) * 100,
      current: processed,
      total: items.length
    })}\n\n`
  }

  yield `data: ${JSON.stringify({ status: 'complete' })}\n\n`
}, {
  body: t.Object({
    items: t.Array(t.String())
  })
})
```

**Using with file streaming:**

```typescript
app.get('/download/:file', async function* ({ params }) {
  const filePath = `./uploads/${params.file}`
  const file = Bun.file(filePath)

  if (!await file.exists()) {
    throw new Error('File not found')
  }

  // Stream file in chunks
  const stream = file.stream()
  for await (const chunk of stream) {
    yield chunk
  }
})
```

Reference: [ElysiaJS Streaming](https://elysiajs.com/essential/handler#streaming)
