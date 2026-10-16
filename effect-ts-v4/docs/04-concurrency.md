# Concurrency and Scheduling

## Contents

- Structured fibers
- Mutable state and coordination
- Queues and PubSub
- Pools and caches
- Retry and repeat schedules

## Fiber ownership

A fiber runs an Effect concurrently. Keep its lifetime attached to an owner rather than treating background work as fire-and-forget.

| API | Lifetime |
|-----|----------|
| `Effect.forkChild(effect)` | Child of the current fiber; interrupted when that parent terminates |
| `Effect.forkScoped(effect)` | Attached to the current scope |
| `Effect.forkDetach(effect)` | Detached from normal parent interruption; requires an explicit shutdown plan |
| `Fiber.join(fiber)` | Await and propagate the fiber's result or failure |
| `Fiber.await(fiber)` | Await its Exit as a value |
| `Fiber.interrupt(fiber)` | Interrupt and await completion |

`Fiber` is not itself an Effect. Join it explicitly. A scope-owned worker survives the acquisition fiber that created it, but not the scope that owns the service.

```ts
import { Effect, Fiber } from "effect"

const program = Effect.gen(function*() {
  const worker = yield* Effect.forkChild(
    Effect.sleep("10 millis").pipe(Effect.as(42))
  )
  return yield* Fiber.join(worker)
})
```

`FiberHandle` manages one current fiber, `FiberMap` manages keyed fibers and `FiberSet` manages a changing set. Use their scoped constructors to clean up active work. Read the manager's replacement behaviour before using the same key for a new job.

Use `Effect.forEach` / `Effect.all` with bounded concurrency for independent work. Use `Semaphore` when several operations must share a concurrency limit. `Effect.raceAll` finds the first success, while `raceAllFirst` returns the first completion, even a failure.

## Refs and coordination

| Primitive | Choose when |
|-----------|-------------|
| `Ref` | Atomic synchronous read/modify/write |
| `SubscriptionRef` | State updates must also form a stream of changes |
| `Deferred` | One producer completes a result awaited by many consumers |
| `Semaphore` | Limit concurrent access to a resource or section |
| `Latch` | Open/close a shared gate |
| Transactional refs/collections | Several values must change together; see [Transactions](07-transactions.md) |

```ts
import { Deferred, Effect, Ref } from "effect"

const program = Effect.gen(function*() {
  const count = yield* Ref.make(0)
  yield* Ref.update(count, (n) => n + 1)
  const previous = yield* Ref.getAndUpdate(count, (n) => n + 1)

  const ready = yield* Deferred.make<string>()
  yield* Deferred.succeed(ready, "ready")
  const status = yield* Deferred.await(ready)
  return { previous, current: yield* Ref.get(count), status }
})
```

`Ref.modify` atomically returns a separate result and replaces the stored value using `[result, newValue]`. Its callback is synchronous. A `get` followed by an effectful operation and `set` is not an atomic read/modify/write; use a semaphore or a suitable service design if the critical section needs effects.

`Deferred` is not directly yieldable; use `Deferred.await`. Completion is one-shot. Decide who completes it on failure or cancellation to avoid waiters that never finish.

`SubscriptionRef.changes` is a Stream including the current value followed by updates. Keep subscriptions scoped, particularly when wiring UI consumers. Use ordinary Ref when change events are unnecessary.

## Semaphore

```ts
import { Effect, Semaphore } from "effect"

const program = Effect.gen(function*() {
  const permits = yield* Semaphore.make(4)
  const request = (id: number) => Effect.succeed(id)
  return yield* Effect.forEach([1, 2, 3, 4, 5], (id) =>
    permits.withPermit(request(id)), { concurrency: "unbounded" })
})
```

Place the semaphore in a shared service if the limit should apply across callers. Creating one per request does not enforce a global limit. `withPermit` releases the permit even on failure or interruption.

## Queue

Queues distribute each element to one taker. Choose capacity and overflow policy deliberately:

| Constructor | At capacity |
|-------------|-------------|
| `Queue.bounded(capacity)` | Producers wait for space |
| `Queue.dropping(capacity)` | Drop the newly offered element |
| `Queue.sliding(capacity)` | Remove old elements to make space |
| `Queue.unbounded()` | Retain elements until consumed; requires an external bound |

Use `Queue.offer` for effectful producers and `Queue.take` for consumers. The queue has both an element type and a completion/error channel. `Queue.end` marks normal completion; `Queue.fail` marks failure. Do not encode normal completion as an arbitrary sentinel unless it belongs in the domain protocol.

```ts
import { Cause, Effect, Queue, Stream } from "effect"

const program = Effect.gen(function*() {
  const queue = yield* Queue.unbounded<number, Cause.Done>()
  yield* Queue.offer(queue, 1)
  yield* Queue.offer(queue, 2)
  yield* Queue.end(queue)
  return yield* Stream.fromQueue(queue).pipe(Stream.runCollect)
})
```

The unbounded queue is intentional in this finite example. Filling a bounded queue before starting its consumer can deadlock. A real producer/consumer pair should run concurrently under the same scope.

Use `Queue.offerUnsafe` only in synchronous callback integration where you cannot run an Effect directly. Choose and document the overflow behaviour there.

## PubSub

PubSub broadcasts each publication to every active subscriber. A Queue load-balances instead. `PubSub.subscribe` is scoped so subscriptions are removed at scope closure.

Subscribe before starting a producer when the first message matters: PubSub is not a durable replay log. Bounded PubSub can backpressure a producer when a slow subscriber stops draining. Dropping/sliding variants choose loss over blocking.

See [Streaming](05-streaming.md) for queue, PubSub and callback stream integration.

## Pools and caches

| Module | Purpose |
|--------|---------|
| `Pool` | Reuse a bounded set of scoped resources |
| `Cache` | Share and memoize effectful lookups with capacity/TTL policies |
| `ScopedCache` | Cache values whose acquisition has finalizers |
| `RcRef` / `RcMap` | Share a scoped resource while it is referenced |

Put caches in layers so callers share the same cache. Include tenant, permissions and other result-affecting context in cache keys where necessary. Decide whether failures should be cached, and use explicit invalidation for writes rather than assuming TTL alone maintains consistency.

A scoped cache entry cannot safely be used after its resource scope closes. Return domain values rather than leaking the underlying acquired handle to a longer-lived caller.

## Schedules

Schedules produce recurring patterns for retrying failures, repeating successes and polling. `Effect.retry(schedule)` re-executes on typed failure; `Effect.repeat(schedule)` re-executes on success. Make repeated side effects idempotent or give them a stable operation key.

| Constructor/combinator | Meaning |
|------------------------|---------|
| `Schedule.recurs(n)` | Permit n recurrences |
| `Schedule.spaced(duration)` | Fixed delay |
| `Schedule.exponential(duration)` | Exponentially growing delay |
| `Schedule.jittered` | Randomise delay to spread load |
| `Schedule.max([a, b])` | Continue while all schedules continue; use the slowest delay |
| `Schedule.min([a, b])` | Continue while any schedule continues; use the fastest delay |
| `Schedule.while(({ input }) => predicate(input))` | Continue only for selected inputs |
| `Schedule.tap(meta => effect)` | Observe retries or repeats |

Cap the delay and the number of retries separately:

```ts
import { Effect, Schedule, Schema } from "effect"

class RemoteError extends Schema.TaggedError<RemoteError>()("RemoteError", {
  retryable: Schema.Boolean
}) {}

declare const request: Effect.Effect<string, RemoteError>

const delay = Schedule.min([
  Schedule.exponential("200 millis"),
  Schedule.spaced("5 seconds")
]).pipe(Schedule.jittered)

const policy = Schedule.max([delay, Schedule.recurs(4)]).pipe(
  Schedule.setInputType<RemoteError>(),
  Schedule.while(({ input }) => input.retryable)
)

const retried = request.pipe(Effect.retry(policy))
```

This permits four retries after the initial attempt. A capped backoff alone does not cap attempts. A schedule builder passed to `Effect.retry(($) => $(schedule).pipe(...))` can infer the failure input type.

Test policies with virtual time rather than real sleeps; see [Testing and observability](14-testing-observability.md).
