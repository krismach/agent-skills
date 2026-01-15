---
title: Stream Large Responses
impact: HIGH
impactDescription: reduces memory usage and improves TTFB
tags: streaming, response, memory, performance
---

## Stream Large Responses

Use streaming for large responses to reduce memory usage and improve Time To First Byte (TTFB). This is especially important for file downloads, large JSON arrays, or database exports.

**Incorrect (buffers entire response in memory):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/export', async (c) => {
  const users = db.query('SELECT * FROM users').all() // All 1M users in memory
  const json = JSON.stringify(users) // Entire JSON string in memory
  return c.json(users) // Sends entire response at once
})

app.get('/download/:file', async (c) => {
  const filename = c.req.param('file')
  const file = Bun.file(`./files/${filename}`)
  const data = await file.arrayBuffer() // Entire file in memory
  return c.body(data)
})
```

**Correct (streams response):**

```typescript
import { Hono } from 'hono'
import { stream } from 'hono/streaming'

const app = new Hono()

// Stream database results
app.get('/export', (c) => {
  return stream(c, async (stream) => {
    await stream.write('[') // Start JSON array

    const stmt = db.query('SELECT * FROM users')
    let first = true

    for (const user of stmt.iterate()) {
      if (!first) await stream.write(',')
      await stream.write(JSON.stringify(user))
      first = false
    }

    await stream.write(']') // End JSON array
  })
})

// Stream file downloads using Bun.file()
app.get('/download/:file', async (c) => {
  const filename = c.req.param('file')
  const file = Bun.file(`./files/${filename}`)

  if (!await file.exists()) {
    return c.notFound()
  }

  // Stream the file
  c.header('Content-Type', file.type)
  c.header('Content-Length', file.size.toString())
  c.header('Content-Disposition', `attachment; filename="${filename}"`)

  return c.body(file.stream())
})
```

**Streaming with backpressure handling:**

```typescript
app.get('/stream-data', (c) => {
  return stream(c, async (stream) => {
    const data = fetchLargeDataset()

    for await (const chunk of data) {
      // stream.write() handles backpressure automatically
      await stream.write(chunk)
      await stream.sleep(100) // Optional: rate limiting
    }
  })
})
```

**Server-Sent Events (SSE) for real-time updates:**

```typescript
import { streamSSE } from 'hono/streaming'

app.get('/events', (c) => {
  return streamSSE(c, async (stream) => {
    let id = 0
    while (true) {
      const message = await getNextEvent()
      await stream.writeSSE({
        data: JSON.stringify(message),
        event: 'update',
        id: String(id++),
      })
      await stream.sleep(1000)
    }
  })
})
```

Benefits:
- Constant memory usage (doesn't grow with response size)
- Better TTFB (client gets first bytes immediately)
- Can handle files larger than available RAM
- Supports Range requests for resumable downloads
- Natural backpressure handling

Reference: [Hono Streaming](https://hono.dev/docs/helpers/streaming)
