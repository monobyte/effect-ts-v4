# HTTP APIs and SQL

## Contents

- HTTP clients and response validation
- Schema-first APIs and middleware
- Serving, typed clients and tests
- SQL repositories, model variants and migrations

## HTTP client services

Depend on `HttpClient.HttpClient` from `effect/http`. Provide a concrete implementation, such as `FetchHttpClient.layer`, once through the service's layer.

```ts
import { Effect, Schema } from "effect"
import { FetchHttpClient, HttpClient, HttpClientResponse } from "effect/http"

const Todo = Schema.Struct({ id: Schema.Int, title: Schema.String })

const fetchTodo = Effect.fn("fetchTodo")(function*(id: number) {
  const client = (yield* HttpClient.HttpClient).pipe(HttpClient.filterStatusOk)
  const response = yield* client.get(`https://example.com/todos/${id}`)
  return yield* HttpClientResponse.schemaBodyJson(Todo)(response)
})

const program = fetchTodo(1).pipe(Effect.provide(FetchHttpClient.layer))
```

Decode responses through schemas rather than casting parsed JSON. Keep HTTP/decoding failures typed, or map them into a domain error with the original cause/reason retained.

| Client operation | Use |
|------------------|-----|
| `HttpClient.mapRequest(transform)` | Common base URL, headers and request policy |
| `HttpClientRequest.prependUrl(base)` | Resolve service-relative paths |
| `HttpClientRequest.acceptJson` | Set an Accept header |
| `HttpClient.filterStatusOk` | Fail on non-2xx responses |
| `HttpClient.retryTransient({ schedule, times })` | Bound transient retries |
| `HttpClientRequest.bodyJsonUnsafe(value)` | Attach already-known JSON-compatible data |
| `HttpClientResponse.schemaBodyJson(S)` | Decode a JSON response |

Build requests with `HttpClientRequest` when method-specific convenience calls are insufficient, then execute with the client. Unsafe JSON construction can throw for non-serializable input; validate domain data before that boundary.

Retried writes need idempotency semantics. A lost response does not prove the server did not apply a write. Scope streamed bodies according to the client implementation; do not return a stream whose response resource has already closed.

## Schema-first HTTP APIs

Separate shared protocol definitions from server implementation. The client can then import schemas and endpoints without pulling server-only services into its bundle.

```ts
import { Effect, Schema } from "effect"
import { HttpApi, HttpApiBuilder, HttpApiEndpoint, HttpApiGroup } from "effect/http-api"

class Greeting extends Schema.Class<Greeting>("app/Greeting")({
  message: Schema.String
}) {}

class Greetings extends HttpApiGroup.make("greetings").add(
  HttpApiEndpoint.get("hello", "/hello/:name", {
    params: { name: Schema.String },
    success: Greeting
  })
) {}

class Api extends HttpApi.make("app").add(Greetings) {}

const GreetingsHandlers = HttpApiBuilder.group(Api, "greetings", (handlers) =>
  handlers.handleAll({
    hello: ({ params }) => Effect.succeed(new Greeting({ message: `Hello ${params.name}` }))
  })
)
```

Each endpoint declares method, path, params/query/payload, success and declared errors. Use schema-backed domain errors with an appropriate HTTP status. `HttpApiSchema.status(404)` can annotate a response error; `Schema.TaggedError` also accepts HTTP metadata.

`GET` payloads use query-string encoding. POST/PATCH payloads normally use a body. Use `HttpApiSchema.asText`, multipart and other helpers for explicit content formats. Path parameters are coerced through the string-tree codec; do not add a second incompatible manual parser.

Group `.prefix("/users")` applies a common path prefix. A top-level group puts its endpoints directly on the generated client; ordinary groups appear as `client.group.endpoint(...)`.

## Authentication and middleware

`HttpApiMiddleware.Service` declares middleware requirements, provided services, security schemes and possible errors. Attach middleware to an endpoint or group. A security middleware can decode a bearer credential and provide a CurrentUser service to handlers.

If `requiredForClient: true`, provide its client implementation with `HttpApiMiddleware.layerClient`. The client wrapper receives `{ next, request }`; it can add the token using `HttpClientRequest.bearerToken` before forwarding.

Authenticate and authorise on the server. Client middleware transports credentials; it is not an access-control boundary. Keep token acquisition separate from shared protocol definitions.

## Server, client and in-memory test wiring

Using the Api and handlers above:

```ts
import { NodeHttpServer, NodeRuntime } from "@effect/platform-node"
import { createServer } from "node:http"
import { Layer } from "effect"
import { HttpRouter, HttpServer } from "effect/http"
import { HttpApiBuilder } from "effect/http-api"

const Routes = HttpApiBuilder.layer(Api, { openapiPath: "/openapi.json" }).pipe(
  Layer.provide(GreetingsHandlers)
)

const serverLayer = HttpRouter.serve(Routes).pipe(
  Layer.provide(NodeHttpServer.layer(createServer, { port: 3000 }))
)
const main = Layer.launch(serverLayer).pipe(NodeRuntime.runMain)

// Alternative web-standard handler for a host that owns the listener:
const web = HttpRouter.toWebHandler(Routes.pipe(Layer.provide(HttpServer.layerServices)))
```

The code is a continuation of the API example. Choose the Node listener or the web handler according to the host. `web.handler` is the web-standard request handler; `web.dispose` closes its resources.

`HttpApiClient.make(Api, { transformClient })` creates a typed client. Provide its HTTP client and any required client middleware. `HttpApiScalar.layer(Api, { path: "/docs" })` serves interactive docs from the shared API.

Use `HttpApiTest.groups(Api, ["greetings"])` to build an in-memory client for selected handler groups. Provide handlers and `HttpServer.layerServices`. This exercises request encoding, routing and response decoding without listening on a socket. It is a stronger boundary test than directly invoking the handler function.

## SQL client and schema queries

Depend on `SqlClient.SqlClient` from `effect/sql`. Provide a driver layer such as `PgClient.layerConfig` or `SqliteClient.layer`. Template interpolations become SQL parameters; do not concatenate untrusted values into query text.

```ts
import { Effect, Schema } from "effect"
import { SqlClient, SqlSchema } from "effect/sql"

const User = Schema.Struct({ id: Schema.Int, name: Schema.String })

const loadUsers = Effect.gen(function*() {
  const sql = yield* SqlClient.SqlClient
  const findAll = SqlSchema.findAll({
    Request: Schema.Void,
    Result: User,
    execute: () => sql`SELECT id, name FROM users ORDER BY id`
  })
  return yield* findAll()
})
```

`SqlSchema` helpers validate requests and decode rows. Choose `findAll`, `findOne` or another helper according to cardinality. A missing record is distinct from query/decoding failure; map only the former to a domain not-found error.

Use `sql.withTransaction(effect)` for database atomicity. An `Effect.tx` transaction only coordinates in-memory transactional structures, not SQL. For an external action that must follow a committed database write, consider an outbox rather than executing the remote call inside a retryable transaction.

## Model variants

`Model.Class` from `effect/schema` defines variants for database and JSON boundaries from one field declaration. `SqlModel.makeRepository` derives common operations against the database variants.

| Variant / field helper | Purpose |
|------------------------|---------|
| `Model` / `.insert` / `.update` | Database select, insert and update contracts |
| `.json` / `.jsonCreate` / `.jsonUpdate` | Public API read/create/update contracts |
| `Model.FieldOnly([...])` | Include a field in selected variants |
| `Model.FieldExcept([...])` | Exclude a field from selected variants |
| `Model.Field({ select, json, ... })` | Supply explicit schemas per variant |
| `Model.UuidV4Insert(Id)` | Application-generated insert UUID |
| `Model.DateTimeInsert` / `DateTimeUpdate` | Managed timestamps |

Use the JSON variants in HTTP contracts so callers cannot set internal fields, generated IDs or managed timestamps accidentally. Use `.makeEffect` when constructing variants with generated values; it can use the Effect clock and other services.

`SqlModel.makeRepository(Model, { tableName, spanPrefix, idColumn })` derives insert/update/findById/delete behaviour. Keep custom queries in the same repository service, with explicit result schemas. A model does not automatically make every SQL schema or database dialect identical.

## Migrations and startup

Driver migration modules supply a loader and a layer. During development, `SqliteMigrator.fromRecord` can describe numbered migration effects; production projects normally keep separate migration files and load them with `fromFileSystem`.

Compose the migration layer with `Layer.provideMerge(SqlLayer)` so dependents receive the SQL service after startup work completes. Plan how multiple application instances coordinate migration execution. Keep DDL changes and data backfills reviewable instead of treating schema generation as an automatic deployment policy.
