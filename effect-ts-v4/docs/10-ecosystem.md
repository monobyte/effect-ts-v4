# Ecosystem and Platform Wiring

## Contents

- Imports and package boundaries
- Runtime and server ownership
- RPC, cluster entities and workflows
- Choosing focused guides

## Package boundaries

Core capabilities live in `effect`; platform implementations, database drivers, AI providers and framework bindings remain separate packages. Keep their versions aligned.

| Capability | Import |
|------------|--------|
| Effects, services, layers, Schema, streams | `effect` |
| HTTP client/server/router | `effect/http` |
| Schema-first HTTP APIs | `effect/http-api` |
| GraphQL clients and resolvers | `effect/graphql` |
| SQL client/schema/repository helpers | `effect/sql` |
| SQL/JSON model variants | `effect/schema` |
| RPC | `effect/rpc` |
| CLI | `effect/cli` |
| Child processes | `effect/process` |
| Language models, tools, chat | `effect/ai` |
| Cluster entities | `effect/cluster` |
| Durable workflow primitives | `effect/workflow` |
| Native Otlp exporters | `effect/observability` |
| NDJSON and binary codecs | `effect/encoding` |
| Test services | `effect/testing` |

Other capabilities include persistence, reactivity, sockets, workers and event logs. Prefer an already-used application abstraction before introducing one.

API stability is declared in metadata. `unstable` APIs may change in minor releases; `experimental` APIs may change in patch releases. Stable APIs follow semver. Do not infer stability from an import path.

| Implementation | Package examples |
|----------------|------------------|
| Node / Bun platform | `@effect/platform-node`, `@effect/platform-bun` |
| SQL drivers | `@effect/sql-pg`, `@effect/sql-sqlite-node`, `@effect/sql-mysql2` |
| AI providers | `@effect/ai-openai`, `@effect/ai-anthropic`, `@effect/ai-openai-compat` |
| Existing OpenTelemetry integration | `@effect/opentelemetry` |
| Vitest integration | `@effect/vitest` |
| Framework-specific atom bindings | `@effect/atom-react`, `@effect/atom-solid`, `@effect/atom-vue` |

## Runtime wiring

Use the platform's `runMain` at the executable entry point, and `Layer.launch` for a service graph that should live until shutdown. Supply the actual platform services required by the graph, rather than assuming a Node runtime automatically provides every platform dependency.

For a framework-owned request lifecycle, use `ManagedRuntime`; for serverless HTTP, use `HttpRouter.toWebHandler` and arrange for its `dispose` function to run at the appropriate host shutdown point. Do not instantiate/dispose the whole graph on every warm request unless per-request isolation is intentional.

## RPC

Define the protocol separately from handlers and transport. `Rpc.make` describes payload, success and error schemas; `RpcGroup.make` groups procedures. Handler implementations are layers derived from the group. Clients and servers then share that protocol rather than independently redefining types.

```ts
import { Effect, Schema } from "effect"
import { Rpc, RpcGroup } from "effect/rpc"

class Health extends RpcGroup.make(
  Rpc.make("Ping", { success: Schema.String })
) {}

const handlers = Health.toLayer({
  Ping: () => Effect.succeed("pong")
})
```

This defines handlers, not a listening server. Choose and provide a matching server/client transport and serialization layer. Application errors belong in RPC error schemas. Transport failures and cancellation remain separate concerns. For streaming RPCs, scope the consumer and define what interruption means for server-side work.

## Cluster entities

An entity combines named RPCs with a stable identity. `Entity.make("Name", [rpcDefinitions])` defines it; `entity.toLayer` supplies handlers. Access a client by yielding `entity.client`, then choose an entity ID.

```ts
import { Effect, Ref, Schema } from "effect"
import { Entity, TestRunner } from "effect/cluster"
import { Rpc } from "effect/rpc"

const Counter = Entity.make("Counter", [
  Rpc.make("Increment", { payload: { amount: Schema.Int }, success: Schema.Int })
])

const CounterLayer = Counter.toLayer(Effect.gen(function*() {
  const count = yield* Ref.make(0)
  return Counter.of({
    Increment: ({ payload }) => Ref.updateAndGet(count, (n) => n + payload.amount)
  })
}), { maxIdleTime: "5 minutes" })

const program = Effect.gen(function*() {
  const clientFor = yield* Counter.client
  return yield* clientFor("counter-1").Increment({ amount: 2 })
})
```

The handler's Ref is in-memory state for an active entity. Passivation or a process restart recreates it; entity identity does not make arbitrary state durable.

Messages are volatile by default. `ClusterSchema.Persisted` marks RPC messages for persistence when the runtime has storage configured. Message persistence is not the same as persisting every handler's memory or guaranteeing arbitrary external effects happen once.

Handlers for one entity run sequentially by default. `Rpc.fork` opts a handler into concurrent processing, so assess shared-state invariants before using it.

For local tests, provide `TestRunner.layer` and the entity layer. Production requires a real cluster transport/storage configuration, such as `NodeClusterSocket.layer()` plus a SQL client. Keep deployment policy outside the entity protocol.

## Workflows

Use workflows when execution must be coordinated across process boundaries and recovery, rather than merely running a long-lived fiber. Define input/output/error schemas, an execution identity and explicit activity boundaries. Provide a workflow engine and durable storage appropriate to the deployment.

External side effects need stable idempotency keys or compensation. A workflow replay must not blindly charge a card or create a remote resource again. Keep retry policy, activity identity and result persistence in the workflow design. Do not claim exactly-once external behaviour merely because the workflow state is durable.

Use normal Effect composition for work whose lifetime is one process/request. Adding a workflow engine is a deployment and persistence decision.

## Focused guides

- [HTTP and SQL](11-http-sql.md): HTTP clients, schema-first APIs, database queries, models and migrations.
- [AI](12-ai.md): provider layers, generation, structured output, tools, execution plans and chat loops.
- [Batching and GraphQL](13-batching-graphql.md): RequestResolver, GraphQL protocol, pagination, testing and batch strategies.
- [Testing and observability](14-testing-observability.md): virtual time, shared layers, property tests, logging and telemetry.
- [CLI and child processes](15-cli-processes.md): command construction, shared flags, subprocess lifetimes and streamed output.
