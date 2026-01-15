---
title: Finalize SQLite Statements
impact: MEDIUM
impactDescription: prevents memory leaks in long-running applications
tags: sqlite, memory, cleanup, resources
---

## Finalize SQLite Statements

Explicitly finalize prepared statements when they're no longer needed to free resources. This is especially important for long-running applications.

**Incorrect (potential memory leak):**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

function processUsers() {
  // Creates new statement each time
  for (let i = 0; i < 1000; i++) {
    const stmt = db.prepare('SELECT * FROM users WHERE id = ?')
    const user = stmt.get(i)
    // Statement is never finalized - memory leak!
  }
}

// Call many times over days/weeks
setInterval(processUsers, 1000)
```

**Correct (properly finalized):**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

function processUsers() {
  // Reuse the same statement
  const stmt = db.prepare('SELECT * FROM users WHERE id = ?')

  for (let i = 0; i < 1000; i++) {
    const user = stmt.get(i)
    // Process user...
  }

  // Clean up when done
  stmt.finalize()
}
```

**Better: Reuse statements globally:**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

// Create statements once, reuse forever
const statements = {
  getUser: db.query('SELECT * FROM users WHERE id = ?'),
  insertUser: db.query('INSERT INTO users (name, email) VALUES (?, ?)'),
  updateUser: db.query('UPDATE users SET name = ?, email = ? WHERE id = ?'),
}

// Use throughout application
function getUser(id: number) {
  return statements.getUser.get(id)
}

// Only finalize on shutdown
process.on('SIGTERM', () => {
  statements.getUser.finalize()
  statements.insertUser.finalize()
  statements.updateUser.finalize()
  db.close()
})
```

**Using try/finally for cleanup:**

```typescript
function performDatabaseOperation() {
  const stmt = db.prepare('INSERT INTO logs (message, timestamp) VALUES (?, ?)')

  try {
    for (const log of logs) {
      stmt.run(log.message, log.timestamp)
    }
  } finally {
    // Always finalize, even if error occurs
    stmt.finalize()
  }
}
```

**Transaction pattern with cleanup:**

```typescript
function bulkInsert(users: User[]) {
  const stmt = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)')

  const transaction = db.transaction(() => {
    for (const user of users) {
      stmt.run(user.name, user.email)
    }
  })

  try {
    transaction()
  } finally {
    stmt.finalize()
  }
}
```

**When to finalize:**

```typescript
// DON'T finalize if used frequently
const getUser = db.query('SELECT * FROM users WHERE id = ?')
// Keep alive for lifetime of application

// DO finalize one-time operations
function migrateData() {
  const stmt = db.prepare('UPDATE old_table SET ...')
  try {
    stmt.run()
  } finally {
    stmt.finalize() // Won't use again
  }
}

// DO finalize in request handlers (if not cached)
app.post('/analytics', (c) => {
  const stmt = db.prepare('INSERT INTO events ...')
  try {
    stmt.run(...)
    return c.json({ success: true })
  } finally {
    stmt.finalize()
  }
})
```

Guidelines:
- Use `db.query()` for frequently reused statements (auto-cached)
- Use `db.prepare()` + `finalize()` for one-time operations
- Always finalize in finally blocks
- Create global statement cache for common queries
- Finalize all statements on shutdown

Reference: [Bun SQLite](https://bun.com/docs/runtime/sqlite)
