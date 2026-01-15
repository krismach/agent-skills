---
title: Use Transactions for Bulk Operations
impact: CRITICAL
impactDescription: 100-1000x faster for bulk inserts/updates
tags: sqlite, transactions, database, bulk, performance
---

## Use Transactions for Bulk Operations

Wrap multiple database operations in a transaction for dramatic performance improvements. SQLite is significantly faster with transactions.

**Incorrect (individual commits):**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

const insertUser = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)')

// Slow: Each insert is a separate transaction
for (let i = 0; i < 10000; i++) {
  insertUser.run(`User ${i}`, `user${i}@example.com`)
}
// Takes ~50 seconds
```

**Correct (using transaction):**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

const insertUser = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)')

// Fast: All inserts in single transaction
const insertMany = db.transaction((users: Array<[string, string]>) => {
  for (const [name, email] of users) {
    insertUser.run(name, email)
  }
})

const users = Array.from({ length: 10000 }, (_, i) =>
  [`User ${i}`, `user${i}@example.com`] as [string, string]
)

insertMany(users)
// Takes ~0.5 seconds (100x faster!)
```

**Transaction best practices:**

```typescript
import { Database } from 'bun:sqlite'

const db = new Database('app.db')

// Define operations
const insertOrder = db.prepare('INSERT INTO orders (user_id, total) VALUES (?, ?)')
const updateInventory = db.prepare('UPDATE products SET stock = stock - ? WHERE id = ?')
const insertOrderItem = db.prepare('INSERT INTO order_items (order_id, product_id, quantity) VALUES (?, ?, ?)')

// Wrap in transaction for atomicity
const createOrder = db.transaction((userId: number, items: OrderItem[]) => {
  // Insert order
  const result = insertOrder.run(userId, calculateTotal(items))
  const orderId = result.lastInsertRowid

  // Update inventory and create order items
  for (const item of items) {
    updateInventory.run(item.quantity, item.productId)
    insertOrderItem.run(orderId, item.productId, item.quantity)
  }

  return orderId
})

// Execute atomically - all or nothing
try {
  const orderId = createOrder(123, items)
  console.log(`Order ${orderId} created`)
} catch (error) {
  // All changes rolled back automatically
  console.error('Order creation failed:', error)
}
```

**Manual transaction control:**

```typescript
// For more complex scenarios
try {
  db.exec('BEGIN TRANSACTION')

  // Perform operations
  insertUser.run('Alice', 'alice@example.com')
  updateBalance.run(100, 'alice@example.com')

  // Conditional logic
  if (someCondition) {
    insertLog.run('Special case')
  }

  db.exec('COMMIT')
} catch (error) {
  db.exec('ROLLBACK')
  throw error
}
```

**Batch processing pattern:**

```typescript
const BATCH_SIZE = 1000

const insertBatch = db.transaction((users: User[]) => {
  for (const user of users) {
    insertUser.run(user.name, user.email)
  }
})

// Process large dataset in batches
for (let i = 0; i < allUsers.length; i += BATCH_SIZE) {
  const batch = allUsers.slice(i, i + BATCH_SIZE)
  insertBatch(batch)
}
```

**Immediate transactions for write-heavy workloads:**

```typescript
// Default: DEFERRED transaction (delays lock acquisition)
const deferredTx = db.transaction(() => {
  // Operations...
})

// IMMEDIATE: Acquires write lock immediately
const immediateTx = db.transaction(() => {
  // Operations...
}, { immediate: true })

// Use IMMEDIATE for write-heavy operations to avoid retries
```

Performance comparison:
- **No transaction**: 50-100 ops/sec
- **With transaction**: 10,000-50,000 ops/sec
- **100-1000x improvement** for bulk operations

When to use transactions:
- ✅ Bulk inserts/updates
- ✅ Multi-table operations that must be atomic
- ✅ Any operation that modifies multiple rows
- ❌ Single row operations (overhead not worth it)
- ❌ Read-only queries (no benefit)

Reference: [Bun SQLite Transactions](https://bun.com/docs/runtime/sqlite)
