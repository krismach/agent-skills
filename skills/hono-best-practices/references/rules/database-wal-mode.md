---
title: Enable SQLite WAL Mode
impact: CRITICAL
impactDescription: 10-20x improvement for concurrent reads/writes
tags: sqlite, database, wal, concurrency
---

## Enable SQLite WAL Mode

Write-Ahead Logging (WAL) mode dramatically improves SQLite performance with concurrent readers and writers. This is essential for production applications.

**Incorrect (default journal mode):**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

// Using default DELETE mode - slow with concurrent access
const users = db.query('SELECT * FROM users').all()
```

**Correct (WAL mode enabled):**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

// Enable WAL mode once during initialization
db.exec('PRAGMA journal_mode = WAL')
db.exec('PRAGMA synchronous = NORMAL') // Safe with WAL
db.exec('PRAGMA cache_size = -64000') // 64MB cache
db.exec('PRAGMA temp_store = MEMORY')
db.exec('PRAGMA busy_timeout = 5000') // 5 second timeout

const users = db.query('SELECT * FROM users').all()
```

**Complete initialization pattern:**

```typescript
function initDatabase(path: string): Database {
  const db = new Database(path)

  // Performance optimizations
  db.exec(`
    PRAGMA journal_mode = WAL;
    PRAGMA synchronous = NORMAL;
    PRAGMA cache_size = -64000;
    PRAGMA temp_store = MEMORY;
    PRAGMA busy_timeout = 5000;
    PRAGMA foreign_keys = ON;
  `)

  return db
}

const db = initDatabase('app.db')
```

WAL mode allows:
- Multiple concurrent readers
- Readers don't block writers
- Writers don't block readers
- Better crash recovery

Reference: [SQLite WAL Mode](https://bun.com/docs/runtime/sqlite)
