# Data Types and Domain Values

## Contents

- Option and Result
- Collections and equality
- Domain modelling and matching
- Duration, DateTime and predicates

## Choose a representation

| Need | Representation |
|------|----------------|
| Value may be absent | `Option.Option<A>` |
| Synchronous computation may fail | `Result.Result<A, E>` |
| Effectful computation may fail | `Effect.Effect<A, E, R>` |
| Ordered ordinary collection | Native array, with `Array` helpers as needed |
| Persistent sequence with efficient concatenation | `Chunk` |
| Persistent map/set with Effect equality | `HashMap` / `HashSet` |
| String-keyed object | `Record` |
| Nominal distinction between otherwise identical types | `Brand` or `Schema.brand` |
| Runtime validated model | `Schema.Struct` / `Schema.Class` |

Do not introduce a specialised collection merely because it is available. Choose it for its operational or equality semantics.

## Option

`Option` represents `Some(value)` or `None`. Its tags are `"Some"` and `"None"`; a Some has a `.value`.

```ts
import { Option } from "effect"

const input = Option.fromNullishOr<string | undefined>(undefined)
const normalized = input.pipe(Option.map((value) => value.trim()))
const label = Option.getOrElse(normalized, () => "untitled")

const description = Option.match(Option.some(42), {
  onNone: () => "missing",
  onSome: (value) => `value=${value}`
})
```

Use `Option.some`, `none`, `fromNullishOr`, `fromNullOr`, or `fromUndefinedOr` to construct. Use `map` for pure transforms, `flatMap` for optional transforms and `filter` for a predicate. `getOrUndefined` and `getOrNull` adapt at non-Effect boundaries. `getOrThrow` is a deliberate throwing boundary.

`Option` is yieldable in `Effect.gen`; None becomes a typed missing-element error. For a domain-specific failure, convert explicitly with `Effect.fromNullishOr` or `Effect.fromOption`, then `Effect.mapError`.

## Result

`Result.Result<A, E>` represents synchronous success or failure. Use `Result.succeed(value)` and `Result.fail(error)`, not Either's left/right vocabulary.

```ts
import { Result } from "effect"

const parsePort = (input: string): Result.Result<number, string> => {
  const value = Number(input)
  return Number.isInteger(value) && value > 0 && value <= 65535
    ? Result.succeed(value)
    : Result.fail("Invalid port")
}

const label = Result.match(parsePort("3000"), {
  onFailure: (error) => error,
  onSuccess: (port) => `port=${port}`
})
```

Tags are `"Success"` and `"Failure"`; payloads are `.success` and `.failure`. `map`, `flatMap` and `mapError` transform the respective channels. A Result can be yielded inside `Effect.gen`, preserving its failure type.

Use `Effect.result(effect)` when you need to inspect typed failures as values. Use `Effect.exit(effect)` to preserve defects and interruption as well.

## Arrays, Chunk and records

`Array` helpers support pipeline composition and return immutable-style results without changing the input. Helpers such as `head`, `last` and `findFirst` express possible absence with Option. Native `.map` and `.filter` remain appropriate for straightforward local code.

`Chunk` is a persistent sequence. Use `Chunk.fromIterable`, `Chunk.make`, `Chunk.append`, `Chunk.map` and `Chunk.toReadonlyArray` where persistent concatenation is useful. Do not assume Stream emits Chunk values: the public stream element type is `A`, and `Stream.runCollect` returns an array.

`Record` helpers operate on string-keyed objects. Use a Map when keys are arbitrary objects or when map semantics better express the problem.

## HashMap, HashSet and equality

Hash collections use Effect's `Hash` and `Equal` protocols. Plain objects normally use identity equality. Structurally equal-looking object literals are therefore not automatically interchangeable keys.

```ts
import { Data, HashMap, HashSet, Option } from "effect"

class UserKey extends Data.Class<{ readonly tenant: string; readonly user: string }> {}
const keyA = new UserKey({ tenant: "a", user: "42" })
const keyB = new UserKey({ tenant: "a", user: "42" })
const users = HashMap.make([keyA, "Ada"])
const found = HashMap.get(users, keyB)
const names = HashSet.fromIterable(["Ada", "Lin", "Ada"])
```

Use `Data.Class` when value equality is needed. `Data.TaggedClass` adds a discriminator. Custom Equal implementations must satisfy the hash contract: equal values have equal hashes. Prefer the existing Data constructors to hand-written hashing.

HashMap operations include `get`, `set`, `remove`, `has`, `map` and `entries`. HashSet offers `add`, `remove`, `has`, `union`, `intersection` and `difference`. Results are new persistent values; retain the returned collection.

## Brands

A nominal brand prevents accidentally mixing IDs with the same underlying primitive. `Brand.nominal<T>()` returns a callable constructor; it does not validate runtime input.

```ts
import { Brand, Schema } from "effect"

type AccountId = string & Brand.Brand<"AccountId">
const AccountId = Brand.nominal<AccountId>()
const internalId = AccountId("account-1")

const UserId = Schema.NonEmptyString.pipe(Schema.brand("UserId"))
type UserId = typeof UserId.Type
const validated = Schema.decodeUnknownSync(UserId)("user-1")
```

Schema brands combine validation with a nominal type. A brand name alone does not validate the format: add the necessary string or numeric checks. Use decoding for untrusted input.

## Match, Order and Equivalence

Use `Match.type<Union>()`, `Match.tag` and `Match.exhaustive` for exhaustive matching on tagged unions. Prefer a normal `switch` when it is clearer. Exhaustiveness matters more than the choice of syntax.

`Equivalence.Equivalence<A>` expresses equality; `Order.Order<A>` expresses comparison. Use module-provided instances and `Order.mapInput` to compare objects by a field. Keep sorting rules explicit rather than assuming domain values have a natural global order.

## Duration

Use `Duration` instead of undocumented numeric units. Effect APIs commonly accept strings such as `"250 millis"` or `"5 seconds"`, or numeric milliseconds. Explicit constructors include `Duration.millis`, `seconds` and `minutes`; use `Duration.toMillis` at integration boundaries.

Duration values represent elapsed time, not calendar dates. For local-day scheduling and daylight-saving behaviour, use a zoned DateTime or scheduling abstraction that expresses calendar semantics.

## DateTime and time zones

`DateTime.now` reads Effect's Clock, so tests can control it. `DateTime.make` returns Option; `makeUnsafe` is for inputs whose validity is already known. DateTime operations are immutable.

```ts
import { DateTime, Effect } from "effect"

const deadline = Effect.gen(function*() {
  const now = yield* DateTime.now
  const end = now.pipe(DateTime.add({ hours: 2 }))
  return DateTime.formatIso(end)
})

const localNow = Effect.gen(function*() {
  const now = yield* DateTime.nowInCurrentZone
  return DateTime.formatIso(now)
}).pipe(Effect.provide(DateTime.layerCurrentZoneNamed("Europe/London")))
```

Use `setZoneNamed` when the IANA zone is untrusted; it returns Option. `setZoneNamedUnsafe` assumes a valid name. A zone controls how an instant is displayed; it does not change that instant.

When interpreting a local date/time in a named zone, use `DateTime.makeZoned` / `makeZonedUnsafe` with `timeZone` and `adjustForTimeZone: true`. Omitting the adjustment treats the input differently. Decide how ambiguous or missing daylight-saving times should be handled at the product boundary.

Use UTC ISO strings for interchange and an explicit user/workspace zone for display. Avoid relying on the server machine's local zone.

## Predicates and refinements

Use `Predicate.isObject`, `isString`, `isNumber` and related guards to narrow unknown values. `Predicate.isObject` allows property access; validate individual properties before using them. A primitive guard is not a substitute for a complete boundary schema.

```ts
import { Predicate } from "effect"

const readName = (value: unknown): string | undefined => {
  if (Predicate.isObject(value) && Predicate.isString(value.name)) {
    return value.name
  }
  return undefined
}
```
