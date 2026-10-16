# Application and Resource Patterns

## Contents

- Choosing a resource abstraction
- ManagedRuntime integration
- Keyed resources with LayerMap
- Layer design and domain boundaries

## Choose by ownership

| Need | Prefer |
|------|--------|
| Resource used once within a computation | `Effect.acquireRelease` + `Effect.scoped` |
| Long-lived service with startup/shutdown | `Layer.effect` |
| Startup task without an exported service | `Layer.effectDiscard` |
| One reusable runtime for a framework | `ManagedRuntime` |
| Shared resource while consumers hold it | `RcRef` |
| Shared resource per key while consumers hold it | `RcMap` |
| Service graph per key | `LayerMap` |
| Bounded reusable resources | `Pool` |
| Memoized values with TTL and capacity | `Cache` |
| Memoized acquired resources | `ScopedCache` |
| Many independent lookups resolved together | `RequestResolver`; see [Batching and GraphQL](13-batching-graphql.md) |

Do not store a raw acquired resource in a plain Ref and then close it while callers still use it. Choose an abstraction that tracks usage, or design an explicit lease protocol.

`RcRef.make({ acquire, idleTimeToLive })` and `RcMap.make({ lookup, idleTimeToLive })` keep acquisition and release scoped. Their `get` operations hold a reference for the caller's scope. An idle TTL delays release; it does not make a released handle safe to reuse. Invalidation affects future acquisition, not a licence to close resources beneath active callers.

## ManagedRuntime

Use a managed runtime when a framework, host application or callback API owns the outer lifecycle. Create it once per intended service lifetime; run domain effects through it; dispose it during shutdown.

```ts
import { Context, Effect, Layer, ManagedRuntime } from "effect"

class Greeting extends Context.Service<Greeting, {
  greet(name: string): Effect.Effect<string>
}>()("app/Greeting") {
  static readonly layer = Layer.succeed(Greeting, {
    greet: (name) => Effect.succeed(`Hello ${name}`)
  })
}

const runtime = ManagedRuntime.make(Greeting.layer)

export const handleGreeting = (name: string) => runtime.runPromise(
  Greeting.use((service) => service.greet(name))
)

export const shutdown = () => runtime.dispose()
```

The host calls `shutdown` when it stops accepting work and finishes its shutdown sequence. Do not dispose after each request if the runtime is intended to share services across requests.

Use `runtime.runPromise` for Promise boundaries and `runSync` only for computations that cannot suspend. Resolve domain failures inside the effect into the host's explicit response shape. A rejected Promise is not a typed HTTP response.

When several managed runtimes should share layers, construct one `Layer.makeMemoMapUnsafe()` at the composition boundary and pass it through `{ memoMap }`. Do not accidentally share tenant-specific state merely because the same process hosts multiple runtimes.

## Keyed service graphs

LayerMap manages a service implementation per key, with scoped acquisition and idle release. It is useful for tenant database pools, workspaces or named sessions.

```ts
import { Context, Effect, Layer, LayerMap } from "effect"

class Workspace extends Context.Service<Workspace, {
  readonly id: string
}>()("app/Workspace") {
  static readonly layer = (id: string) => Layer.succeed(Workspace, { id })
}

class Workspaces extends LayerMap.Service<Workspaces>()("app/Workspaces", {
  lookup: (id: string) => Workspace.layer(id),
  idleTimeToLive: "1 minute"
}) {}

const currentWorkspace = Workspace.use((workspace) => Effect.succeed(workspace.id))

const program = currentWorkspace.pipe(
  Effect.provide(Workspaces.get("workspace-1")),
  Effect.provide(Workspaces.layer)
)
```

The real workspace layer can acquire files, subscriptions or a connection with `acquireRelease`; LayerMap then manages those finalizers. `Workspaces.invalidate(key)` requests rebuilding on later access.

Include all configuration affecting an instance in its key or surrounding lifetime. A cache keyed only by tenant ID is insufficient if the same ID can resolve to multiple incompatible configurations in one runtime.

## Layer design

- Separate the service interface from live, in-memory and test implementations.
- Keep `.layerNoDeps` when consumers need to choose dependencies; expose `.layer` for normal assembly.
- Capture expensive clients once in layer construction, then implement methods around those clients.
- Keep request-specific values out of a shared service's mutable state. Pass them as parameters or scoped context.
- Compose dependent layers with `Layer.provide`; merge independent outputs with `Layer.mergeAll`.
- Start owned background workers with scoped fibers, not detached module-level work.

Nested provides may reuse enclosing layer instances; sibling provides do not automatically share new builds. Prefer one visible application graph. Use `Layer.fresh` or `Effect.provide(layer, { local: true })` for deliberate isolation.

## Error and schema boundaries

Use schemas at external boundaries, branded or class-based models where they help, and domain services for behaviour. Preserve the distinction between an absent record, invalid input, a transport failure and a programming defect.

When wrapping a lower-level error, preserve it as a typed `reason` if the caller may recover specifically. Use `cause: Schema.Defect()` for opaque diagnostic causes. Do not blanket-map every failure to `Error(String(error))`; it destroys useful structure.

## Avoid these implementation traps

| Trap | Consequence | Better approach |
|------|-------------|-----------------|
| Run an Effect inside another service with `runPromise` | Escapes requirements, cancellation and scope ownership | Compose with `yield*` / `flatMap` |
| Acquire an HTTP/DB client per call | Repeated startup and fragmented lifetime | Acquire in a service layer |
| Share a mutable test layer across independent tests | State leaks between tests | Fresh per-test provision, or explicitly shared fixtures |
| Promise executor starts before its Effect runs | Work escapes lazy execution | Construct inside `tryPromise` / `promise` |
| Apply a global cache without context-aware keys | Cross-user or cross-tenant results | Include relevant context or isolate instances |
| Restart a stream blindly after partial writes | Duplicate processing | Stable IDs, checkpoints and idempotent writes |
| Perform external effects in an in-memory transaction | Repeated, non-rollbackable effects | Commit local state first or use durable coordination |
