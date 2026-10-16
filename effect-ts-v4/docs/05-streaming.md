# Streaming

## Contents

- Sources and consumption
- Transformation and concurrency
- Callback and platform integration
- Encoding, Sink, Channel and Pull

## Stream values

`Stream.Stream<A, E, R>` describes an incremental source of `A` values with failures `E` and requirements `R`. Construction is lazy. A stream runs when consumed, and scoped resources remain owned by that consumption.

| Source | Constructor |
|--------|-------------|
| Individual values | `Stream.make(a, b)` / `Stream.succeed(a)` |
| Array or iterable | `Stream.fromIterable(values)` |
| One Effect result | `Stream.fromEffect(effect)` |
| Repeated Effect | `Stream.fromEffectRepeat(effect)` |
| Scheduled polling | `Stream.fromEffectSchedule(effect, schedule)` |
| Paginated API | `Stream.paginate(cursor, nextPage)` |
| Async iterable | `Stream.fromAsyncIterable(iterable, mapError)` |
| Web ReadableStream | `Stream.fromReadableStream(...)` |
| Queue or broadcast | `Stream.fromQueue` / `Stream.fromPubSub` |
| DOM events | `Stream.fromEventListener(target, event)` |
| Callback API | `Stream.callback(setup)` |

Use a stream when incremental processing, cancellation or backpressure matters. An array plus `Effect.forEach` is often sufficient for an already-loaded finite collection.

## Consumption

```ts
import { Effect, Sink, Stream } from "effect"

const values = Stream.fromIterable([1, 2, 3, 4]).pipe(
  Stream.filter((n) => n % 2 === 0),
  Stream.mapEffect((n) => Effect.succeed(n * 10), { concurrency: 2 })
)

const collected = Stream.runCollect(values)
const total = values.pipe(Stream.runFold(() => 0, (sum, n) => sum + n))
const sumViaSink = values.pipe(Stream.run(Sink.sum))
```

`Stream.runCollect` returns an array, not a Chunk. Only collect streams that are finite and fit in memory.

| Consumer | Result |
|----------|--------|
| `Stream.runCollect` | All values in an array |
| `Stream.runDrain` | Execute and discard outputs |
| `Stream.runForEach(f)` | Effectful per-element consumption |
| `Stream.runFold(() => initial, step)` | Accumulated value; initial state is a thunk |
| `Stream.runHead` / `runLast` | Option of the first or last value |
| `Stream.run(sink)` | Sink-specific result |

## Transformations

Use `map` for pure transforms, `mapEffect` for effectful work and `flatMap` when each element produces another Stream. Concurrency options belong on the effectful or flattening operation. Bound them according to the downstream service's capacity.

`filter`, `take`, `drop`, `takeWhile` and related operators narrow a stream. `scan` emits accumulated state; `runFold` emits one final result. Use `concat` for sequential sources, `merge` for concurrent interleaving and `zip` to pair elements. Choose ordering explicitly when switching to concurrent processing.

Error handling mirrors Effect: `Stream.catch`, `catchTag`, `mapError` and `tapError` operate on stream failures. Retrying or recovering by restarting a source can replay previously emitted values; downstream writes need suitable deduplication or idempotency.

## Pagination and polling

`Stream.paginate` returns one page plus an optional next cursor. It flattens the page values and requests the next page as downstream pulls.

```ts
import { Effect, Option, Schedule, Stream } from "effect"

declare const fetchPage: (cursor: number) => Effect.Effect<{
  readonly items: ReadonlyArray<string>
  readonly next: number | undefined
}, Error>

const pages = Stream.paginate(0, (cursor) => fetchPage(cursor).pipe(
  Effect.map((page) => [page.items, Option.fromUndefinedOr(page.next)] as const)
))

const polls = Stream.fromEffectSchedule(
  Effect.succeed("healthy"),
  Schedule.spaced("30 seconds")
).pipe(Stream.take(3))
```

Prefer this to fetching every page before constructing the stream. The latter defeats early termination and backpressure.

## Callback sources

A callback stream exposes a queue. Register listeners inside setup, and attach unregistering to its scope.

```ts
import { Effect, Queue, Stream } from "effect"

const pointerEvents = (target: HTMLElement) =>
  Stream.callback<PointerEvent>(Effect.fnUntraced(function*(queue) {
    const listener = (event: PointerEvent) => {
      Queue.offerUnsafe(queue, event)
    }
    yield* Effect.acquireRelease(
      Effect.sync(() => target.addEventListener("pointermove", listener)),
      () => Effect.sync(() => target.removeEventListener("pointermove", listener))
    )
  }))
```

For high-rate callbacks, explicitly choose buffering and overflow policy. A native event emitter cannot usually suspend to respect backpressure. Unbounded buffering can grow without limit.

Use `Stream.fromEventListener` instead when no custom setup is needed. For Node streams use `NodeStream.fromReadable` from `@effect/platform-node`, with lazy evaluation and an explicit error mapper. Do not acquire a platform stream outside Effect and accidentally make it live before consumption.

## Encoding and framed data

Use channels to decode data that may cross arbitrary input chunk boundaries. Splitting each network chunk manually does not reliably split lines, UTF-8 code points or binary frames.

```ts
import { Schema, Stream } from "effect"
import { Ndjson } from "effect/encoding"

const Event = Schema.Struct({
  id: Schema.Int,
  message: Schema.String
})

const decoded = Stream.make('{"id":1,"message":"started"}\n').pipe(
  Stream.pipeThroughChannel(Ndjson.decodeSchemaString(Event)()),
  Stream.runCollect
)
```

| Format | Operations |
|--------|------------|
| UTF-8 bytes/text | `Stream.decodeText()` / `Stream.encodeText` |
| Text lines | `Stream.splitLines` |
| NDJSON strings | `Ndjson.decodeString()` / `encodeString()` |
| Schema-validated NDJSON | `Ndjson.decodeSchemaString(schema)()` / `encodeSchemaString(schema)()` |
| NDJSON bytes | Corresponding non-String Ndjson variants |
| Schema-derived framed binary | `SchemaBinary.decode(schema)()` / `encode(schema)()` from `effect/encoding` |

Schema-aware encoding applies transformations, such as DateTime to ISO text, before serializing.

## Sink

A Sink consumes a stream and returns a result, possibly leaving input for later consumption. Prefer common sinks such as `Sink.sum`, `count`, `drain` and `forEach` before building a custom consumer. Use `Stream.run` to apply a Sink.

Do not copy a v3 Sink type signature into new code. The current implementation uses the same pull/channel model as streams, with completion represented separately from ordinary failure.

## Channel and Pull

Use Channel when implementing a reusable protocol transform, codec or lower-level stream primitive. It represents transformation between an upstream pull and a downstream pull. `Stream.pipeThroughChannel` applies it without requiring application code to manage the protocol manually.

A Pull is an Effect-like request for the next output, with `Cause.Done` marking normal completion. End-of-stream is not an application failure. `Pull.catchDone` handles that completion deliberately; generic error recovery must not turn every completion into an infinite retry.

Use `Stream.toPull` only within a scope and drive it until completion. Prefer regular Stream operators for application logic; they already handle cancellation, completion and resource lifetime.
