# Transactions

## Contents

- Transaction boundaries
- Retrying on changed state
- Transactional collections
- Side effects and failure behaviour

## Atomic in-memory changes

Use `Effect.tx` to commit coordinated changes to transactional data structures. Transactions use the normal Effect type, so ordinary generator composition and typed errors work inside them.

```ts
import { Effect, TxRef } from "effect"

const program = Effect.gen(function*() {
  const available = yield* TxRef.make(10)
  const reserved = yield* TxRef.make(0)

  yield* Effect.tx(Effect.gen(function*() {
    yield* TxRef.update(available, (n) => n - 1)
    yield* TxRef.update(reserved, (n) => n + 1)
  }))

  return yield* Effect.tx(Effect.gen(function*() {
    return {
      available: yield* TxRef.get(available),
      reserved: yield* TxRef.get(reserved)
    }
  }))
})
```

Successful completion commits the journal together. Typed failure, defects or interruption prevent the journal from committing. Concurrent modifications to accessed transactional values can cause the body to run again.

Nested `Effect.tx` calls join the existing transaction and reuse its journal and retry state. The outermost call commits or rolls back the composed operation. A nested call does not create an independently committed transaction.

Wrap related reads in a transaction too if they must observe one consistent snapshot. Reading several refs separately does not give that invariant.

## Waiting for a condition

Use `Effect.txRetry` to suspend until relevant state changes. Read the relevant transactional value before retrying, so the transaction knows what can wake it.

```ts
import { Effect, Schema, TxRef } from "effect"

class InvalidAmount extends Schema.TaggedError<InvalidAmount>()("InvalidAmount", {
  amount: Schema.Number
}) {}

const transfer = Effect.fn("transfer")(function*(
  source: TxRef.TxRef<number>,
  target: TxRef.TxRef<number>,
  amount: number
) {
  if (!Number.isFinite(amount) || amount <= 0) {
    return yield* new InvalidAmount({ amount })
  }
  yield* Effect.tx(Effect.gen(function*() {
    const balance = yield* TxRef.get(source)
    if (balance < amount) return yield* Effect.txRetry
    yield* TxRef.set(source, balance - amount)
    yield* TxRef.update(target, (value) => value + amount)
  }))
})
```

This waits for funds rather than failing with an insufficient-funds error. That is a domain choice. Fail explicitly instead if the caller expects an immediate rejection. Apply a timeout outside the transaction if the wait must be bounded.

`Effect.txRetry` is transactional coordination, not a retry schedule for a remote call. `Effect.retry` re-executes failures according to a policy; it is a different mechanism.

## Transactional data structures

| Module | Use |
|--------|-----|
| `TxRef` | One mutable transactional value |
| `TxChunk` | Transactional ordered collection |
| `TxHashMap` | Transactional keyed state |
| `TxHashSet` | Transactional membership |
| `TxQueue` | Transactional producer/consumer coordination |
| `TxSemaphore` | Transactional permit accounting |

For a TxRef, `make`, `get`, `set`, `update` and `modify` provide the common operations. Like Ref.modify, `modify` returns `[result, newValue]` from its callback. Use a single transaction around multi-collection invariants, not one transaction per individual write.

Do not mix normal Ref writes into a transaction and assume they roll back. Only transactional values participate in the journal.

## Side effects

A transaction body may run multiple times. Keep network calls, SQL writes, file writes, queue publications and user-visible logging outside it. They are not undone when a transaction retries or fails.

Return a value describing what committed, then perform the external action afterward. If an external action must be guaranteed alongside persistent state, use a database transaction and an outbox or another durable coordination design. `Effect.tx` does not make a database, remote API or distributed entity atomic with local memory.

Transaction metadata is available through `Effect.Transaction` inside the boundary, including attempt and retry information. Use it for diagnostics when needed, without introducing side effects that assume exactly-once execution.

See [HTTP and SQL](11-http-sql.md) for `sql.withTransaction`, which coordinates actual database operations.
