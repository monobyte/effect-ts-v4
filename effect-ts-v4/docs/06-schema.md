# Schema: Validation, Parsing and Serialization

## Contents

- Schema and codec types
- Domain models and parsers
- Optional fields, defaults and checks
- Transformations and tagged errors
- Interoperability, optics and custom types

## Type and encoded representation

Import `Schema` from `effect`. A schema describes both the decoded value and how it crosses a boundary. `Schema.Schema<A>` describes the decoded type; `Schema.Codec<A, I, RD, RE>` also tracks encoded representation `I`, decoding requirements `RD` and encoding requirements `RE`.

Do not describe all schemas as `Schema<A, I, R>`: the current API separates these views. Prefer concrete inference unless an API really needs a generic constraint.

```ts
import { Schema } from "effect"

const User = Schema.Struct({
  id: Schema.Int,
  name: Schema.NonEmptyString,
  role: Schema.Literals(["admin", "member"])
})

type User = typeof User.Type
type UserEncoded = typeof User.Encoded
```

Use `typeof S.Type` / `typeof S.Encoded`, or indexed-access equivalents, to extract representations. Decoding services and encoding services can differ when transformations use effects.

## Constructors

| Shape | Schema |
|-------|--------|
| Primitives | `String`, `Number`, `Boolean`, `BigInt`, `Unknown`, `Never` |
| Finite or integral numbers | `Finite`, `Int`, `Natural` |
| One literal | `Literal("open")` |
| Several literals | `Literals(["open", "closed"])` |
| Alternatives | `Union([A, B])` |
| Record with known fields | `Struct({ field: schema })` |
| String-keyed map | `Record(Schema.String, valueSchema)` |
| Fixed-length positional data | `Tuple([A, B])` |
| Sequence | `Array(itemSchema)` / `NonEmptyArray(itemSchema)` |
| Nullable / undefined value | `NullOr(S)` / `UndefinedOr(S)` |
| Optional property | `optional(S)` / `optionalKey(S)` |
| Validated class | `Class<Self>("identifier")({ fields })` |

`Schema.Number` permits values that a wire format or domain may reject. Choose Finite or explicit checks when NaN and infinities are invalid.

## Domain classes and decoding

```ts
import { Effect, Schema } from "effect"

class User extends Schema.Class<User>("app/User")({
  id: Schema.Int,
  name: Schema.NonEmptyString,
  role: Schema.Literals(["admin", "member"])
}) {}

const decodeUser = Schema.decodeUnknownEffect(User)
const encodeUser = Schema.encodeEffect(User)

const parseUser = Effect.fn("parseUser")((input: unknown) => decodeUser(input))
```

Reuse decoders instead of reconstructing them for every request. Parse unknown input at boundaries, then pass validated domain values internally.

Schema classes can be constructed with `new User(fields)`. Current schema values also expose `.make`, `.makeEffect` and `.makeUnsafe` for their construction protocols. Defaults and effectful construction may require `.makeEffect`; use the inferred input type and do not assume it equals the decoded type. Unsafe construction is not a replacement for decoding untrusted input.

Do not extend `Schema.Struct` as if it were a class. Use a Struct constant with an inferred type, or `Schema.Class`.

| Need | API |
|------|-----|
| Typed Effect validation | `Schema.decodeUnknownEffect(S)(input)` |
| Synchronous boundary | `Schema.decodeUnknownSync(S)(input)`; throws on failure |
| Promise host | `Schema.decodeUnknownPromise(S)(input)` |
| Pure success/failure value | `Schema.decodeUnknownResult(S)(input)` |
| Optional success | `Schema.decodeUnknownOption(S)(input)` |
| Encode known value in Effect | `Schema.encodeEffect(S)(value)` |
| Encode unknown input | `Schema.encodeUnknownEffect(S)(input)` |
| Runtime type guard | `Schema.is(S)` |
| Assertion | `Schema.asserts(S, input)` |

Choose an execution variant compatible with the schema's service requirements. Do not use Sync merely to bypass effects a codec needs.

## Checks

Use `Schema.check` with reusable checks such as `isPattern`, `isGreaterThan` or a custom `Schema.makeFilter`. Use `Schema.refine` for a TypeScript refinement that narrows the decoded type.

```ts
import { Schema } from "effect"

const Slug = Schema.String.pipe(
  Schema.check(Schema.isPattern(/^[a-z0-9]+(?:-[a-z0-9]+)*$/, {
    message: "Expected a lowercase URL slug"
  })),
  Schema.brand("Slug")
)
```

Checks validate, transformations change representation. A brand distinguishes types; it does not supply a missing check. Attach error descriptions to the relevant field or check, so callers receive useful boundary errors.

## Optionality, null and defaults

These represent different contracts:

| Definition | Contract |
|------------|----------|
| `field: Schema.String` | Key and string value required |
| `field: Schema.optionalKey(Schema.String)` | Key may be absent; a present value must be a string |
| `field: Schema.optional(Schema.String)` | Key may be absent or explicitly undefined |
| `field: Schema.NullOr(Schema.String)` | Key required; value may be null |

Use `Schema.withDecodingDefault(Effect.succeed(value))` for a default on the decoding side. Other helpers distinguish a missing key, a missing value and constructor defaults. Pick the helper that expresses the actual API contract instead of treating all absence as the same thing.

For PATCH-style APIs, preserve absent versus explicitly null fields. For SQL-backed models, consider variant schemas with `Model.Class`; see [HTTP and SQL](11-http-sql.md).

## Bidirectional transformations

Prefer built-in codecs such as `Schema.NumberFromString` and `Schema.DateTimeUtcFromString`. For a custom codec, pipe the source into `Schema.decodeTo(target, transformation)`.

```ts
import { Schema, SchemaGetter } from "effect"

const BooleanFromText = Schema.Literals(["yes", "no"]).pipe(
  Schema.decodeTo(Schema.Boolean, {
    decode: SchemaGetter.transform((text) => text === "yes"),
    encode: SchemaGetter.transform((value) => value ? "yes" as const : "no" as const)
  })
)
```

The custom decode maps the source's decoded value to the target's encoded input; encode maps in the reverse direction. Both directions must be valid. Use `SchemaGetter.transformEffect` when mapping can fail or needs services; the service requirements become part of the codec.

`Schema.toType(S)` validates the decoded side without repeating the original wire transformation. `Schema.toEncoded(S)` describes the encoded side. Do not accidentally decode an already-decoded value with a codec that expects a string.

## Tagged errors

```ts
import { Schema } from "effect"

class InvalidInput extends Schema.TaggedError<InvalidInput>()("InvalidInput", {
  field: Schema.String,
  message: Schema.String
}) {}

const error = new InvalidInput({ field: "name", message: "Required" })
```

Use `Schema.TaggedError` for errors that should have schema validation and serialization. `Data.TaggedError` remains available for ordinary typed error classes; it is not universally prohibited. Choose the schema-backed form when the error crosses HTTP, RPC, workflow or persistence boundaries.

For nested failures, a `reason` field can itself be a tagged schema union. See [Core Effect](01-core-effect.md) for reason-based recovery.

## Error handling and interoperability

Parsing failures use `Schema.SchemaError`, carrying structured issues. Map them into application errors at the boundary, preserving details needed for diagnostics. Avoid returning raw untrusted input or secret values as an error response merely because the issue contains them.

- `Schema.toStandardSchemaV1(S)` adapts to Standard Schema consumers.
- `Schema.toJsonSchemaDocument(S)` produces a JSON Schema document, including reusable definitions.
- `Schema.toRepresentation(S)` exposes the schema representation used for conversions.
- `Schema.toFormatter(S)` derives formatting; `Schema.toEquivalence(S)` derives equality.
- `Arbitrary` derives generators for property tests; see [Testing and observability](14-testing-observability.md).

Choose decoded versus encoded representation intentionally when generating a wire contract. An in-memory DateTime, class or branded type may not itself be the JSON representation.

## Optics and internals

`Schema.toIso(S)` derives an optic between the schema's decoded type and its Iso view. `Optic` supports focused immutable reads and updates. Use optics when a reusable focus improves clarity; direct object updates remain appropriate for simple changes.

`SchemaAST` describes the syntax tree. `SchemaIssue`, `SchemaGetter` and `SchemaTransformation` support custom parsing and transformation behaviour. Prefer public constructors before manually constructing AST nodes. When declaring a custom type with `Schema.declare`, provide accurate validation, annotations and any representation required by JSON Schema or arbitrary generation; an opaque runtime check alone cannot magically describe all those outputs.
