# Migrating Effect v3 to v4

## Contents

- Migration order and package imports
- Services and runtime ownership
- Core API and semantic changes
- Schema, streams and transactions
- Verification checklist

## Migration order

Migrate in dependency order: packages and imports, service identifiers, layer construction, schema/error boundaries, computation APIs, then host runtimes. Type-check after each stage. Avoid broad replacements when the new operation has different semantics.

All Effect ecosystem packages share a version. Upgrade platform, SQL driver, AI provider, telemetry and testing packages alongside `effect`.

## Imports

| Previous package or path | Current path |
|--------------------------|--------------|
| `@effect/platform` HTTP modules | `effect/http` / `effect/http-api` |
| `@effect/rpc` | `effect/rpc` |
| `@effect/sql` | `effect/sql` |
| `@effect/cli` | `effect/cli` |
| `@effect/ai` | `effect/ai` |
| `@effect/cluster` | `effect/cluster` |
| `@effect/workflow` | `effect/workflow` |
| Schema from a separate package | `Schema` from `effect` |
| TestClock in the root package | `TestClock` from `effect/testing` |

Capability imports use `effect/<capability>`. The import path does not contain an `unstable` segment. Stability is declared in API metadata; moving a module does not imply its API is stable. Platform implementations and provider/driver packages remain separate.

## Services and context

| v3 pattern | v4 pattern |
|------------|------------|
| `Context.GenericTag<Shape>(id)` | `Context.Service<Shape>(id)` |
| `Context.Tag(id)<Self, Shape>()` | `Context.Service<Self, Shape>()(id)` |
| `Effect.Tag` / `Effect.Service` | Explicit `Context.Service` plus implementation layers |
| Fiber-local configurable values | `Context.Reference` / built-in `References` |
| Implicit generated default layer | Explicit static `.layer` and dependency composition |

```ts
import { Context, Effect, Layer } from "effect"

class ClockLabel extends Context.Service<ClockLabel, {
  readonly label: Effect.Effect<string>
}>()("app/ClockLabel") {
  static readonly layer = Layer.succeed(ClockLabel, {
    label: Effect.succeed("UTC")
  })
}
```

Use `Service["Service"]` to extract its implementation type. Do not assume the old generated accessors, `.Default`, or `.Live` conventions appear automatically.

`Layer.effect` supports scoped acquisition. Replace `Layer.scoped` constructions accordingly. Context-producing layer constructors are `succeedContext` and `effectContext`.

Nested `Effect.provide` calls can reuse layers built by an enclosing provide/runtime. Sibling provides still build their own copies unless sharing is explicit. Use `{ local: true }` to isolate a layer subtree, or `Layer.fresh` for a fresh build.

## Core APIs

| v3 name/pattern | Current form |
|-----------------|--------------|
| `Effect.catchAll` | `Effect.catch` |
| `Effect.catchAllCause` | `Effect.catchCause` |
| `Effect.fork` | `Effect.forkChild` |
| `Effect.forkDaemon` | `Effect.forkDetach` |
| `Effect.zipRight` | `Effect.andThen` |
| `Effect.either` / Either values | `Effect.result` / Result values |
| `Either.right` / `Either.left` | `Result.succeed` / `Result.fail` |
| Yield Ref, Deferred or Fiber directly | `Ref.get`, `Deferred.await`, `Fiber.join` |
| Recursive Cause tree | `cause.reasons` and reason helpers |
| `Effect.gen(this, generator)` | `Effect.gen({ self: this }, generator)` |
| Nested helper type namespaces | Check flattened module helpers such as `Effect.Success<T>` |

`Config` remains yieldable but is not an Effect subtype. Compose with Config combinators, or read inside a generator before using Effect operations.

`Effect.firstSuccessOf` is still sequential fallback. Do not replace it with `Effect.raceAll`, which executes concurrently. `raceAllFirst` returns the first completion including failure. These are behavioural choices, not interchangeable renames.

Use object-form `Effect.tryPromise({ try, catch })` for explicit error mapping. Ensure `Effect.tap` callbacks return an Effect. Keep existing error and cancellation semantics while updating syntax.

## Schema

| v3 pattern | v4 pattern |
|------------|------------|
| `Schema.Schema<A, I, R>` | Separate Schema/Codec views; infer concrete schemas where possible |
| `Schema.Schema.Type<typeof S>` | `typeof S.Type` |
| `Schema.decodeUnknown` / `encode` Effect variants | `decodeUnknownEffect` / `encodeEffect` |
| `Schema.Union(A, B)` | `Schema.Union([A, B])` |
| `Schema.Literal("a", "b")` | `Schema.Literals(["a", "b"])` |
| `Schema.Tuple(A, B)` | `Schema.Tuple([A, B])` |
| `Schema.Record({ key, value })` | `Schema.Record(key, value)` |
| `filter(predicate)` | `check(makeFilter(predicate))` |
| `filter(refinement)` | `refine(refinement)` |
| `transform` / `transformOrFail` | `decodeTo` with SchemaGetter/SchemaTransformation |
| `asserts(schema)(input)` | `asserts(schema, input)` |
| Extend Struct as a class | Struct constant + inferred type, or `Schema.Class` |

Check wire formats, not just TypeScript errors. `Schema.Date` expects a Date value; use a string-to-Date codec when the input is ISO text. Many `*FromSelf` schemas dropped that suffix. Optional keys, undefined and null are distinct contracts; choose `optionalKey`, `optional` and `NullOr` deliberately.

Use `Schema.TaggedError<Self>()("Tag", fields)` for schema-backed errors. `Data.TaggedError` is still valid for ordinary error classes; do not mechanically convert every Data error because an old guide says its generic form is prohibited.

Schema transformations track decoding and encoding service requirements separately. Test both directions, defaults, excess-property behaviour, optional fields and failure formatting.

## Streams and transactions

`Stream.runCollect` returns an array. `Stream.runFold` takes an initial-state thunk. Update low-level Sink/Channel/Pull code against the current pull model rather than transplanting old generic signatures.

Transactional work uses the normal Effect type with `Effect.tx` as the boundary, `Effect.txRetry` to wait for state changes and `Tx*` collections for journaled state. Nested boundaries compose. Keep external side effects outside retryable bodies; a local transaction does not roll back SQL or network actions.

## Runtimes and logging

Use `ManagedRuntime` for repeated calls from a host framework, and dispose it on shutdown. Keep platform `runMain` at executable boundaries. Rework old Runtime-specific integration around these ownership boundaries rather than recreating private runtime internals.

Log levels are string values such as `"Info"` and `"Error"`. Provide custom loggers through `Logger.layer` instead of removed replacement helpers. Prefer native Otlp exporters for new telemetry integrations; preserve an existing OpenTelemetry setup through `@effect/opentelemetry` when appropriate.

## Verification checklist

- Align package versions and inspect actual export paths.
- Type-check service definitions and composed layer requirements.
- Test expected failures, defects, interruption and finalizers.
- Check sequential versus concurrent behaviour and retry attempt limits.
- Decode real boundary payloads and encode them back to the expected wire format.
- Verify resource sharing and test isolation after layer memoization changes.
- Test stream completion, backpressure and replay/idempotency.
- Run existing tests and the production build; inspect bundle changes where relevant.

Use the other topic guides for current examples rather than carrying compatibility shims into new code.
