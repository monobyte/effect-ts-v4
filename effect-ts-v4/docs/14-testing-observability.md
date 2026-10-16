# Testing and Observability

## Contents

- Effect tests and virtual time
- Layer substitution and isolation
- Boundary and property tests
- Logging, tracing and metrics

## Effect tests

Use `@effect/vitest` to run effects with test services and resource cleanup. Import TestClock from `effect/testing`, not the root package.

```ts
import { assert, it } from "@effect/vitest"
import { Effect, Fiber } from "effect"
import { TestClock } from "effect/testing"

it.effect("finishes after its delay", () => Effect.gen(function*() {
  const fiber = yield* Effect.forkChild(
    Effect.sleep("1 minute").pipe(Effect.as("ready"))
  )
  yield* TestClock.adjust("1 minute")
  assert.strictEqual(yield* Fiber.join(fiber), "ready")
}))
```

Fork time-dependent work before advancing virtual time. Sleeping in the main test fiber and waiting for real time defeats the test clock. Use `it.live` only when the test intentionally needs real runtime services.

Use `it.effect.each(cases)` for parameterized tests. Inspect typed failures with `Effect.flip` or `Effect.result`; use `Effect.exit` for defects and interruption. Assert the error's tag/fields rather than matching a stringified failure trace.

## Test services and layers

Provide a complete in-memory implementation when it clarifies the behaviour under test. Keep mutable state inside the layer acquisition, not a module-level Map shared by every test.

```ts
import { assert, it } from "@effect/vitest"
import { Context, Effect, Layer, Ref } from "effect"

class Counter extends Context.Service<Counter, {
  readonly increment: Effect.Effect<number>
}>()("test/Counter") {}

const TestCounter = Layer.effect(Counter, Effect.gen(function*() {
  const state = yield* Ref.make(0)
  return Counter.of({ increment: Ref.updateAndGet(state, (n) => n + 1) })
}))

it.effect("starts with a fresh counter", () => Effect.gen(function*() {
  const counter = yield* Counter
  assert.strictEqual(yield* counter.increment, 1)
}).pipe(Effect.provide(TestCounter, { local: true })))
```

`layer(AppLayer)("suite", it => ...)` creates one shared layer for that test block and disposes it after the suite. That is useful for expensive resources and intentional integration fixtures; state persists between its tests. Reset explicitly or use per-test provision when independence is required.

`Layer.provideMerge` exposes dependencies to tests that also need to inspect them. `Layer.provide` hides them from the resulting output. Do not weaken the production service interface just to add testing accessors.

Test finalizers and cancellation for resources or background workers: cancel a consumer and verify the subscription/worker closes. A happy-path method assertion does not establish lifecycle correctness.

## Boundary tests

| Boundary | Test approach |
|----------|---------------|
| Pure domain service | In-memory dependency layers |
| HTTP API encode/router/decode | `HttpApiTest.groups` with handlers and `HttpServer.layerServices` |
| GraphQL business logic | `GraphQLClient.layerTest` |
| GraphQL transport, middleware and partial errors | Real client over scripted GraphQLProtocol |
| SQL queries and migrations | Actual driver with temporary test database |
| Retry, timeout and polling | TestClock plus controlled failure/success effects |
| Streaming/callback integration | Cancellation, completion, buffering and cleanup cases |

Tests should cover the contract at the relevant boundary. A GraphQL stub does not validate schemas; an in-memory repository does not validate SQL.

## Property tests

Use schema-derived generators to test properties, not a second implementation of the same algorithm.

```ts
import { assert, it } from "@effect/vitest"
import { Effect, Schema } from "effect"

it.effect.prop("reversing twice preserves a string", [Schema.String], ([value]) =>
  Effect.sync(() => {
    assert.strictEqual(value.split("").reverse().reverse().join(""), value)
  })
)
```

The current `Arbitrary` module supports native schema-derived generation. Prefer `it.effect.prop` for Effect properties. When moving older property tests, revisit generator options and replay behaviour rather than assuming fast-check option names carry over unchanged.

A failed random example should have enough replay information to reproduce it. Add targeted cases for rare conditions such as empty input, very large values, overflow and daylight-saving boundaries; random sampling alone may miss them.

## Structured logging

Effect logging is contextual and structured. Log levels are strings such as `"Debug"`, `"Info"`, `"Warn"` and `"Error"`.

```ts
import { Effect, Layer, Logger, References } from "effect"

const Logging = Logger.layer([Logger.consoleJson]).pipe(
  Layer.provideMerge(Layer.succeed(References.MinimumLogLevel, "Warn"))
)

const program = Effect.logWarning("Inventory below threshold").pipe(
  Effect.annotateLogs({ service: "orders", warehouse: "west" }),
  Effect.withLogSpan("inventory-check"),
  Effect.provide(Logging)
)
```

Use `Logger.layer([...])` to configure loggers. `Logger.toFile` requires a platform filesystem implementation. `Logger.batched` supports a flush window and effectful flush callback; keep it scoped so shutdown flushes and closes resources.

Use stable event names and structured fields. Avoid logging credentials, full prompts, personal data or raw request payloads by default. Redacted configuration is only redacted until it is unwrapped.

## Tracing

`Effect.fn("Service.method")` creates useful method spans. Add nested spans with `Effect.withSpan` and attributes with `Effect.annotateCurrentSpan` / `annotateSpans`. Keep labels low-cardinality; IDs belong in attributes when useful, not in span names.

For new integrations, the native Otlp exporters avoid requiring the full OpenTelemetry SDK:

```ts
import { Layer } from "effect"
import { FetchHttpClient } from "effect/http"
import { OtlpLogger, OtlpSerialization, OtlpTracer } from "effect/observability"

const resource = { serviceName: "orders", serviceVersion: "1.0.0" }

const Telemetry = Layer.mergeAll(
  OtlpTracer.layer({ url: "http://localhost:4318/v1/traces", resource }),
  OtlpLogger.layer({ url: "http://localhost:4318/v1/logs", resource })
).pipe(
  Layer.provide(OtlpSerialization.layerJson),
  Layer.provide(FetchHttpClient.layer)
)
```

Provide telemetry outside the application service graph so acquisition spans are captured too. Ensure the application's shutdown closes the telemetry scope to flush pending exports. Use `@effect/opentelemetry` when integrating into an existing SDK or instrumentation setup rather than installing two competing tracing pipelines.

## Metrics

Use Metric counters for totals, gauges for current state and histograms for distributions such as latency. Keep dimensions bounded: tenant IDs, request IDs and raw URLs can create unbounded cardinality.

Choose the exporter required by the deployment; defining a Metric does not publish it automatically. Test service behaviour independently of a live telemetry collector, and test exporter wiring separately where the integration matters.
