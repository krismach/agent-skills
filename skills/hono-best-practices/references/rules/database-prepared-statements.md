---
title: Use Prepared Statements
impact: CRITICAL
impactDescription: 3-6x faster than better-sqlite3, prevents SQL injection
tags: sqlite, database, prepared, security, performance
---

## Use Prepared Statements

Bun's SQLite driver caches prepared statements automatically, providing massive performance gains. Always use parameterized queries for security and speed.

**Incorrect (string interpolation - dangerous and slow):**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

// DANGEROUS: SQL injection vulnerability
function getUser(id: number) {
  return db.query(`SELECT * FROM users WHERE id = ${id}`).get()
}

// Slow: parses query every time
for (let i = 0; i < 1000; i++) {
  db.query(`INSERT INTO logs (message) VALUES ('Log ${i}')`).run()
}
```

**Correct (prepared statements with parameters):**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

// Safe and fast: query is cached automatically
const getUser = db.query('SELECT * FROM users WHERE id = ?')

// Reuse the prepared statement
const user1 = getUser.get(1)
const user2 = getUser.get(2)

// Batch inserts efficiently
const insertLog = db.query('INSERT INTO logs (message) VALUES (?)')
for (let i = 0; i < 1000; i++) {
  insertLog.run(`Log ${i}`)
}
```

**Using transactions for bulk operations:**

```typescript
const insertUser = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)')

// Wrap in transaction for atomic operations
const insertMany = db.transaction((users: Array<[string, string]>) => {
  for (const [name, email] of users) {
    insertUser.run(name, email)
  }
})

// Execute transaction
insertMany([
  ['Alice', 'alice@example.com'],
  ['Bob', 'bob@example.com'],
  ['Charlie', 'charlie@example.com'],
])
```

**Cleaning up when completely done:**

```typescript
const stmt = db.prepare('SELECT * FROM users WHERE age > ?')

// Use the statement many times
const adults = stmt.all(18)
const seniors = stmt.all(65)

// Explicitly finalize when you're done forever
stmt.finalize()
```

Benefits:
- Protection against SQL injection
- Query parsing happens once
- Automatic caching in Bun
- 3-6x faster than better-sqlite3
- Type-safe with proper typing

Reference: [Bun SQLite Performance](https://bun.com/docs/runtime/sqlite)
