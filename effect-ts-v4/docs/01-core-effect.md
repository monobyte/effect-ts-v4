# Core Effect

## Contents

- Effect values and composition
- Typed errors, defects and reasons
- Resource scopes and cancellation
- Running programs

## Effect values

`Effect.Effect<A, E, R>` describes a computation producing `A`, failing with a typed error `E`, and requiring services `R`. Creating an Effect does not run it. Keep effects as values until an application boundary.

| Source | Constructor | Failure behaviour |
|--------|-------------|-------------------|
| Existing value | `Effect.succeed(value)` | Cannot fail |
| Synchronous side effect | `Effect.sync(() => value)` | Thrown exceptions become defects |
| Fallible synchronous API | `Effect.try({ try, catch })` | Map exceptions to a typed error |
| Promise that should not reject | `Effect.promise(() => promise)` | Rejections become defects |
| Fallible Promise API | `Effect.tryPromise({ try, catch })` | Map rejections to a typed error |
| Callback API | `Effect.callback(resume => finalizer)` | Resume with an Effect; return cancellation cleanup |
| Nullable value | `Effect.fromNullishOr(value)` | Fails when absent |

Prefer explicit error mapping for external APIs. Honour the supplied `AbortSignal` when wrapping cancellable Promise APIs.

```ts
import { Effect, Schema } from "effect"

class ReadFailed extends Schema.TaggedError<ReadFailed>()("ReadFailed", {
  cause: Schema.Defect()
}) {}

const readText = Effect.fn("readText")((url: string) =>
  Effect.tryPromise({
    try: async (signal) => {
      const response = await fetch(url, { signal })
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      return response.text()
    },
    catch: (cause) => new ReadFailed({ cause })
  })
)
```

`Effect.try` also accepts a thunk with an `UnknownError` failure, but explicit domain errors usually give callers a more useful contract.

## Composition

Use `Effect.gen` for inline imperative control flow. Use `Effect.fn("name")` for reusable functions that should create tracing spans. Use `Effect.fnUntraced` when tracing is unnecessary, particularly in hot library paths. Avoid functions whose only purpose is wrapping `Effect.gen`.

```ts
import { Effect } from "effect"

const normalize = Effect.fn("normalize")(function*(input: string) {
  yield* Effect.logDebug("Normalizing input")
  return input.trim().toLowerCase()
})

const program = Effect.gen(function*() {
  const name = yield* normalize(" Ada ")
  yield* Effect.logInfo(name)
  return name
})
```

| Operation | Use |
|-----------|-----|
| `Effect.map(f)` | Transform a successful value synchronously |
| `Effect.flatMap(f)` | Choose the next Effect from a successful value |
| `Effect.andThen(next)` | Sequence another computation |
| `Effect.as(value)` / `Effect.asVoid` | Replace or discard the successful value |
| `Effect.tap(f)` | Run an Effect while keeping the original success |
| `Effect.all(effects, options)` | Combine effects while preserving the input structure |
| `Effect.forEach(items, f, options)` | Traverse with explicit concurrency |
| `Effect.suspend(() => effect)` | Defer construction until execution |

`tap` callbacks return an Effect. Wrap synchronous side effects in `Effect.sync`; do not pass a void-returning logger directly.

`Effect.all` and `forEach` are sequential by default. Set `{ concurrency: 8 }` for bounded parallel work. Use `"unbounded"` only when the workload permits it. Do not replace sequential fallback with a race:

| API | Behaviour |
|-----|-----------|
| `Effect.firstSuccessOf(effects)` | Try effects in order until one succeeds; if all fail, return the last error |
| `Effect.raceAll(effects)` | Run concurrently; first success wins, interrupting remaining competitors |
| `Effect.raceAllFirst(effects)` | Run concurrently; first completion wins, including failure |

## Typed errors and defects

Expected failures belong in `E`; unexpected failures use the defect channel. Do not indiscriminately convert infrastructure errors to defects to satisfy a service type.

```ts
import { Effect, Schema } from "effect"

class MissingUser extends Schema.TaggedError<MissingUser>()("MissingUser", {
  id: Schema.String
}) {}

const lookup = Effect.fn("lookup")(function*(id: string) {
  if (id.length === 0) return yield* new MissingUser({ id })
  return { id }
})

const recovered = lookup("").pipe(
  Effect.catchTag("MissingUser", (error) =>
    Effect.succeed({ id: `fallback:${error.id}` })
  )
)
```

Return when yielding a terminal failure so TypeScript understands that execution stops. Schema tagged errors are yieldable. `Effect.fail(error)` is also explicit and works with ordinary error values.

| API | Scope |
|-----|-------|
| `Effect.catch(handler)` | All typed failures |
| `Effect.catchTag(tag, handler)` | One `_tag` in the error union |
| `Effect.catchTags({ Tag: handler })` | Several tagged errors |
| `Effect.mapError(f)` | Transform typed errors |
| `Effect.tapError(f)` | Observe typed failure without recovery |
| `Effect.catchCause(handler)` | Full cause, including defects and interruption |
| `Effect.exit(effect)` | Capture the full outcome as `Exit` |
| `Effect.result(effect)` | Capture typed success/failure as `Result`; defects still propagate |
| `Effect.flip(effect)` | Swap typed failure and success, useful in tests |
| `Effect.orDie` | Turn typed failure into a defect deliberately |

`Cause` is flat: inspect `cause.reasons`, not a recursive Sequential/Parallel tree. `Exit` represents success or a failure cause; it preserves defects and interruption that `Result` does not represent.

## Errors with reasons

Use a tagged `reason` field when a domain error wraps several recoverable causes. This keeps the outer API stable without throwing away the specific reason.

```ts
import { Effect, Schema } from "effect"

class RateLimited extends Schema.TaggedError<RateLimited>()("RateLimited", {
  retryAfter: Schema.Finite
}) {}
class Rejected extends Schema.TaggedError<Rejected>()("Rejected", {
  message: Schema.String
}) {}
class RequestFailed extends Schema.TaggedError<RequestFailed>()("RequestFailed", {
  reason: Schema.Union([RateLimited, Rejected])
}) {}

declare const request: Effect.Effect<string, RequestFailed>

const recovered = request.pipe(
  Effect.catchReason("RequestFailed", "RateLimited", (reason) =>
    Effect.succeed(`Retry after ${reason.retryAfter}`)
  )
)

const unwrapped = request.pipe(
  Effect.unwrapReason("RequestFailed"),
  Effect.catchTags({
    RateLimited: (reason) => Effect.succeed(`Wait ${reason.retryAfter}`),
    Rejected: (reason) => Effect.succeed(reason.message)
  })
)
```

Use `Effect.catchReasons("OuterTag", { ReasonTag: handler })` for several reasons at once. An optional fallback handles other reasons; without it they remain typed failures.

## Resource scopes

Acquire resources with `Effect.acquireRelease`; release runs on success, failure or interruption when the owning scope closes. `Effect.scoped` supplies and closes a scope. `Effect.ensuring` attaches cleanup to one computation. Use `Effect.addFinalizer` for cleanup attached to an existing scope.

```ts
import { Effect } from "effect"

const program = Effect.scoped(Effect.gen(function*() {
  const resource = yield* Effect.acquireRelease(
    Effect.sync(() => ({ closed: false })),
    (resource) => Effect.sync(() => { resource.closed = true })
  )
  yield* Effect.logInfo(`Resource open: ${!resource.closed}`)
}))
```

Acquire a long-lived client inside its service layer rather than on every service method call. `Layer.effect` can acquire scoped resources; the layer owns their lifetime.

## Cancellation and timeouts

Interruption is cooperative and propagates through structured fibers. Use an uninterruptible region only around a small invariant-preserving operation. Prefer scoped cleanup over trying to manually catch every cancellation.

`Effect.timeout` imposes a time limit and introduces a typed timeout failure. Retry only failures that are safe to repeat; see [Concurrency](04-concurrency.md) for schedules and fiber ownership.

## Running programs

| Boundary | Runner |
|----------|--------|
| Promise-based host or request handler | `Effect.runPromise` |
| Need the full result including failure cause | `Effect.runPromiseExit` |
| Entirely synchronous computation | `Effect.runSync` |
| Need a fiber handle | `Effect.runFork` |
| Node command or server | `NodeRuntime.runMain` from `@effect/platform-node` |
| Bun command or server | `BunRuntime.runMain` from `@effect/platform-bun` |
| Layer whose lifetime is the program | `Layer.launch(layer)` then the platform runner |
| Framework owns application lifecycle | `ManagedRuntime`; see [Patterns](08-patterns.md) |

Runners require a fully provided environment. Keep them out of reusable service methods and libraries. Promise runners reject on failure; do not use rejection text as a domain error protocol.
