# Services, Layers and Configuration

## Contents

- Service definitions and references
- Layer construction and composition
- Memoization and resource lifetime
- Configuration

## Service definitions

A service identifier is a typed key; its implementation is an ordinary value. Use `Context.Service`, a package-qualified identifier, and a static `.layer` for the normal implementation.

```ts
import { Context, Effect, Layer, Ref } from "effect"

class Counter extends Context.Service<Counter, {
  readonly get: Effect.Effect<number>
  increment: Effect.Effect<void>
}>()("app/Counter") {
  static readonly layer = Layer.effect(Counter, Effect.gen(function*() {
    const value = yield* Ref.make(0)
    return Counter.of({
      get: Ref.get(value),
      increment: Ref.update(value, (n) => n + 1)
    })
  }))
}

const program = Effect.gen(function*() {
  const counter = yield* Counter
  yield* counter.increment
  return yield* counter.get
}).pipe(Effect.provide(Counter.layer))

type CounterService = Counter["Service"]
```

`yield* Counter` adds the identifier `Counter` to the requirement type, not the implementation shape. `Counter.of` checks an implementation without building or running a layer.

For a simple key without static members, use `Context.Service<Shape>("key")`. Prefer the class form for application services that expose layers and associated helpers.

Keep service interfaces focused on domain operations. Expose typed failures in methods, rather than leaking low-level transport or database details unless callers need them. Use `Effect.fn` for implementation methods where spans are useful.

## Context and defaults

`Context.Context<R>` stores service implementations. `Context.make(Key, value)` creates one entry; `Context.add`, `get`, `getOption`, `merge` and `mergeAll` operate on the immutable context. Most application wiring should use layers instead of constructing contexts manually.

Use `Context.Reference` for a value with a meaningful default:

```ts
import { Context, Effect } from "effect"

const FeatureEnabled = Context.Reference<boolean>("app/FeatureEnabled", {
  defaultValue: () => false
})

const readFlag = Effect.gen(function*() {
  return yield* FeatureEnabled
})

const enabled = readFlag.pipe(Effect.provideService(FeatureEnabled, true))
```

References need no implementation in the required environment. Overrides are local to the provided effect and inherited by child fibers. Use them for contextual policy, not mutable application storage. Use `Ref` or a service for mutable state.

`References` contains built-in runtime configuration values. Use documented reference keys rather than reaching into fiber internals. See [Concurrency](04-concurrency.md) for fiber-local behaviour.

## Layer construction

`Layer.Layer<ROut, E, RIn>` builds services `ROut`, may fail with `E`, and requires services `RIn`.

| Constructor | Use |
|-------------|-----|
| `Layer.succeed(Service, implementation)` | Existing implementation |
| `Layer.sync(Service, () => implementation)` | Lazily create a synchronous implementation |
| `Layer.effect(Service, effect)` | Effectful acquisition, including scoped resources |
| `Layer.effectDiscard(effect)` | Startup/shutdown work without a service output |
| `Layer.succeedContext(context)` | Existing multi-service context |
| `Layer.effectContext(effect)` | Effect that produces a context |
| `Layer.unwrap(effectOfLayer)` | Choose a layer from configuration or other effects |
| `Layer.empty` | No services |

Use `Layer.effectDiscard` for migrations, background workers or startup validation that must complete before dependent layers start. It does not invent a dummy service to represent that work.

## Composition

Read the input and output service types when composing. A pipe's visual order is not enough to determine which services remain available.

| Expression | Meaning |
|------------|---------|
| `Layer.mergeAll(A, B)` | Build independent layers; expose both outputs |
| `B.pipe(Layer.provide(A))` | Use A to satisfy B; expose B's outputs |
| `B.pipe(Layer.provideMerge(A))` | Use A to satisfy B; expose both outputs |
| `Layer.provide([A, B])` | Provide several dependency layers |
| `effect.pipe(Effect.provide(AppLayer))` | Build and provide the final service graph |

```ts
import { Context, Effect, Layer } from "effect"

class Prefix extends Context.Service<Prefix, string>()("app/Prefix") {}
class Greeter extends Context.Service<Greeter, {
  greet(name: string): Effect.Effect<string>
}>()("app/Greeter") {
  static readonly layerNoDeps = Layer.effect(Greeter, Effect.gen(function*() {
    const prefix = yield* Prefix
    return Greeter.of({ greet: (name) => Effect.succeed(`${prefix} ${name}`) })
  }))
  static readonly layer = this.layerNoDeps.pipe(
    Layer.provide(Layer.succeed(Prefix, "Hello"))
  )
}
```

A useful convention is `.layerNoDeps` for an implementation that leaves dependencies visible and `.layer` for the normal assembled implementation. Test layers can substitute dependencies without altering service code.

## Dynamic selection

Use `Layer.unwrap` to read configuration before choosing an implementation:

```ts
import { Config, Context, Effect, Layer } from "effect"

class Endpoint extends Context.Service<Endpoint, string>()("app/Endpoint") {}

const EndpointLayer = Layer.unwrap(Effect.gen(function*() {
  const local = yield* Config.Boolean("USE_LOCAL").pipe(Config.withDefault(false))
  const url = local ? "http://localhost:3000" : yield* Config.String("API_URL")
  return Layer.succeed(Endpoint, url)
}))
```

Do not hide required dependencies in a module-level runner. Let `Layer.unwrap` preserve configuration errors and remaining requirements in the resulting layer type.

## Memoization and lifetime

Layers memoize by layer identity inside their memoization scope. Reuse the same layer value to share a service; calling a factory twice can produce distinct layers and therefore distinct resources. `Layer.fresh` opts out of sharing deliberately.

Nested `Effect.provide` calls can reuse the enclosing layer memo map. Sibling calls do not share newly built instances automatically. Do not assume that a repeated provide always constructs a fresh service. For test isolation, choose an explicit fresh layer or a new managed runtime rather than relying on incidental wiring.

Use `Effect.acquireRelease` inside `Layer.effect` for sockets, subscriptions, database clients and similar resources. Their finalizers run when the layer's owning scope closes. Starting unmanaged fibers in a constructor bypasses that ownership.

Use `Layer.mock` for partial test implementations only when unused members should fail if called. A complete explicit test implementation is usually easier to reason about.

## Configuration

`Config` is yieldable, but is not itself an `Effect`. Compose it with `Config` combinators, or read it inside `Effect.gen` before using Effect combinators.

```ts
import { Config, Effect, Redacted } from "effect"

const config = Effect.gen(function*() {
  const port = yield* Config.Int("PORT").pipe(Config.withDefault(3000))
  const production = yield* Config.Boolean("PRODUCTION").pipe(Config.withDefault(false))
  const apiKey = yield* Config.Redacted("API_KEY")
  return { port, production, apiKey }
})
```

Common constructors are `Config.String`, `Int`, `Number`, `Boolean`, `URL`, `Duration` and `Redacted`. Use `Config.all` to assemble records, `Config.map` for pure transforms, `Config.nested` for namespacing, `Config.option` for optional values and `Config.withDefault` for deliberate defaults.

`Config.Redacted` keeps the value out of ordinary formatting. Unwrap with `Redacted.value` only at the integration boundary. Redaction does not prevent a caller from deliberately logging the unwrapped value.

`ConfigProvider` controls where configuration comes from. Environment variables are the normal default; `ConfigProvider.fromUnknown` supplies structured test/configuration data. Provide an explicit provider to isolate tests from the host process. Consult the installed provider API for key-path conventions when mixing nested objects and environment names.

Use [Patterns](08-patterns.md) for `ManagedRuntime` integration and [Testing and observability](14-testing-observability.md) for dependency substitution and test layers.
