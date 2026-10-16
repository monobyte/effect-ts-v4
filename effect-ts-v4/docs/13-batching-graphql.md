# Request Batching and GraphQL

## Contents

- RequestResolver and batching policy
- GraphQL operation contracts and clients
- Pagination, partial results and retries
- Ordered and aliased batching
- Test boundaries

## RequestResolver

A Request describes one logical lookup. A resolver collects lookups into batches, invokes the data source and completes each request. Keep it inside a service layer so callers share the resolver and its cache instead of constructing one per call.

```ts
import { Effect, Exit, Request, RequestResolver } from "effect"

class GetName extends Request.Class<
  { readonly id: number },
  string,
  never,
  never
> {}

const program = Effect.gen(function*() {
  const resolver = yield* RequestResolver.make<GetName>(
    Effect.fnUntraced(function*(entries) {
      for (const entry of entries) {
        entry.completeUnsafe(Exit.succeed(`user-${entry.request.id}`))
      }
    })
  ).pipe(
    RequestResolver.setDelay("5 millis"),
    RequestResolver.withCache({ capacity: 256 })
  )

  const getName = (id: number) => Effect.request(new GetName({ id }), resolver)
  return yield* Effect.forEach([1, 2, 1], getName, { concurrency: "unbounded" })
})
```

The example uses a stub lookup. For a remote source, issue one bulk request for the collected keys, then map results back to every entry. Complete each entry with `Exit.succeed(value)` or `Exit.fail(error)`; do not leave missing entries waiting forever.

`entry.context` carries a request's requirements. Do not silently use one request's auth or tenant context for every entry in a batch. Group by the relevant context, or isolate resolvers, when the upstream call cannot safely mix requests.

| Policy | Effect |
|--------|--------|
| `RequestResolver.setDelay(duration)` | Collect more requests at the cost of initial latency |
| `RequestResolver.batchN(n)` | Limit the batch size to what the source supports |
| `RequestResolver.withCache({ capacity })` | Share repeated lookup results according to resolver cache semantics |
| `RequestResolver.withSpan(name)` | Trace resolver execution and request links |

`withCache` caches completed successes and failures by request equality; interrupted results are not cached. Its LRU/FIFO capacity does not imply a TTL: entries have no time-based expiration. Include tenant/auth-sensitive fields in request equality or isolate the resolver.

For TTL, refresh or explicit invalidation, use `RequestResolver.asCache(resolver, { capacity, timeToLive })`, then normal `Cache.get`, `Cache.invalidate` and `Cache.refresh` operations. Its `requireServicesAt` option chooses construction-time versus lookup-time requirements. A cache hit must not bypass an authorisation decision that can change between callers.

Concurrent calls create a batching opportunity. A loop that awaits each call before starting the next generally cannot combine them into one batch. Keep concurrency bounded where input sizes or remote limits require it.

## GraphQL contracts

Import `GraphQL`, `GraphQLGroup`, `GraphQLClient`, `GraphQLProtocol`, `GraphQLMiddleware` and `GraphQLResolver` from `effect/graphql`.

An operation declares its document, variable schemas and result schema. A group defines the client's operations. Protocol and authentication are layers, so business code depends on a typed client service.

```ts
import { Context, Layer, Schema } from "effect"
import { GraphQL, GraphQLClient, GraphQLGroup, GraphQLProtocol } from "effect/graphql"
import { FetchHttpClient } from "effect/http"

const Product = GraphQL.query("Product", {
  document: "query Product($id: ID!) { product(id: $id) { id name } }",
  variables: { id: Schema.String },
  result: Schema.Struct({
    product: Schema.NullOr(Schema.Struct({ id: Schema.String, name: Schema.String }))
  })
})
const Operations = GraphQLGroup.make(Product)

class Catalog extends Context.Service<
  Catalog,
  GraphQLClient.FromGroup<typeof Operations>
>()("app/Catalog") {
  static readonly layerTest = GraphQLClient.layerTest(Catalog, Operations)
  static readonly layer = Layer.effect(Catalog, GraphQLClient.make(Operations)).pipe(
    Layer.provide(GraphQLProtocol.layerHttp({ url: "https://example.com/graphql" })),
    Layer.provide(FetchHttpClient.layer)
  )
}
```

For a non-trivial schema, generate operation modules from `.graphql` files using `@effect/graphql-generator`. Use the generated variable and result schemas instead of manually duplicating large selections. Nullability is part of the contract; do not cast nullable selections to required values.

Attach authentication with `GraphQLMiddleware.Service` and a group's `.middleware(Auth)`. `GraphQLMiddleware.mapRequest` can add headers. Required middleware becomes a dependency of `GraphQLClient.make`; provide it at the client layer boundary.

Queries and mutations use HTTP POST with the HTTP protocol. Subscriptions can use graphql-sse or a WebSocket protocol. Choose the transport supported by the endpoint.

## Pagination and errors

`GraphQLClient.pages(operation, options)` streams pages. `GraphQLClient.items(operation, { variables, connection })` drives a cursor connection and streams its items. The connection accessor must locate the pageInfo/nodes shape. Downstream demand controls page fetching.

By default, a GraphQL response containing `errors` fails even when it includes data. Pass `{ partial: true }` as the operation's second argument when the caller intentionally needs partial data plus errors. Handle both; do not silently present incomplete data as complete.

`GraphQLClientError` retains structured reasons, HTTP status, response headers and retry information. `reason._tag === "ResponseError"` identifies GraphQL response failures; provider-specific error fields remain available in each error entry.

Queries and mutations are not automatically retried. Inspect `isRetryable` and honor `retryAfter`; some APIs also require an `x-ratelimit-reset` time when remaining quota reaches zero. Bound the retry count. A mutation is retryable only if its application semantics allow resubmission.

Apply page-request retries inside middleware or at the individual request boundary. Retrying the entire pagination stream restarts at its first page and can duplicate items already processed. Subscriptions have a reconnect policy; choose an explicit policy for non-retryable limits or authentication failures.

## GraphQL batch strategies

| Upstream capability | Resolver |
|---------------------|----------|
| Bulk input, results in corresponding order | `GraphQLResolver.ordered` |
| Bulk input, identify returned rows by key | `GraphQLResolver.findById` |
| Single-item field, no bulk endpoint | `GraphQLResolver.aliased` |

`GraphQLResolver.request(resolver)` creates the lookup function; callers can use it as an ordinary Effect-returning service method.

For `ordered(client.Nodes, { variables: ids => ({ ids }), path: ["nodes"], typename: "Product" })`, the source must return results in the promised order. Null entries or nonmatching typenames can become missing-element failures. Use `RequestResolver.batchN` for the endpoint's maximum IDs per call.

`aliased(client.Operation)` builds one query with separately aliased root selections and renamed variables. It decodes each slice separately; errors under one root alias fail that slice instead of necessarily failing unrelated calls.

Aliasing requires the generator's structured `GraphQL.DocumentTemplate`, not an arbitrary raw document string. Preserve generated documents rather than attempting fragile string replacement of variables and selections. Its default batch size is ten slices; adjust with `batchN` based on query cost and server limits.

If middleware requires per-request context such as CurrentUser, provide a `batchKey` that separates different users/tenants. The common operation name alone is not a safe batching key across auth contexts.

## Testing

`GraphQLClient.layerTest(Service, Group)` returns a helper accepting operation-name handlers. Handlers receive decoded variables and return decoded results. Missing handlers fail with an UnimplementedError defect.

That test layer bypasses transport, encoding, decoding and middleware. Use it for domain/service behaviour. For partial data plus errors, wire a real `GraphQLClient.make` over a scripted `GraphQLProtocol` layer. Use protocol-level or live integration tests for wire compatibility and authentication, rather than claiming a service stub covers those paths.
