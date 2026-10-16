# CLI and Child Processes

## Contents

- Typed commands, flags and arguments
- Shared flags and subcommands
- Process collection and pipelines
- Streaming output and shutdown

## Commands

Import `Command`, `Argument` and `Flag` from `effect/cli`. A command describes inputs and a handler returning an Effect. Provide platform services and run it at the executable boundary.

```ts
import { NodeRuntime, NodeServices } from "@effect/platform-node"
import { Console, Effect, Schema } from "effect"
import { Argument, Command, Flag } from "effect/cli"

const greet = Command.make("greet", {
  name: Argument.String("name").pipe(Argument.withSchema(Schema.NonEmptyString)),
  loud: Flag.Boolean("loud").pipe(Flag.withAlias("l"), Flag.withDefault(false))
}, ({ name, loud }) => Console.log(loud ? name.toUpperCase() : name)).pipe(
  Command.withDescription("Print a greeting")
)

const main = greet.pipe(
  Command.run({ version: "1.0.0" }),
  Effect.provide(NodeServices.layer)
)

NodeRuntime.runMain(main)
```

Use schemas to validate parsed values before the handler sees them. Keep handler effects reusable so tests can exercise business logic separately from command parsing.

| Input feature | API |
|---------------|-----|
| String / boolean flag | `Flag.String(name)` / `Flag.Boolean(name)` |
| Enumerated choices | `Flag.Literals(name, ["low", "high"])` |
| Optional value | `Flag.optional` |
| Default | `Flag.withDefault(value)` |
| Alias | `Flag.withAlias("v")` |
| Domain validation | `Flag.withSchema(S)` / `Argument.withSchema(S)` |
| Help description | `withDescription(text)` |
| Usage examples | `Command.withExamples([{ command, description }])` |

Optional flags become Option values; inspect them with Option operations rather than assuming undefined.

## Shared flags and subcommands

`Command.withSharedFlags` makes selected root flags available to subcommands. A subcommand can yield the root command to access its parsed shared input.

```ts
import { Console, Effect } from "effect"
import { Command, Flag } from "effect/cli"

const root = Command.make("tasks").pipe(
  Command.withSharedFlags({
    workspace: Flag.String("workspace").pipe(Flag.withDefault("personal"))
  })
)

const list = Command.make("list", {}, Effect.fnUntraced(function*() {
  const { workspace } = yield* root
  yield* Console.log(`Listing tasks in ${workspace}`)
}))

const tasks = root.pipe(Command.withSubcommands([list]))
```

Apply `Command.run` once to the final command tree. Do not independently run every subcommand or manually reparse process arguments in its handler.

## ChildProcess and platform services

A `ChildProcess` is a command description. `ChildProcessSpawner` executes it, and a platform implementation supplies process/filesystem behaviour. For Node, `NodeServices.layer` provides the required services.

```ts
import { NodeServices } from "@effect/platform-node"
import { Effect } from "effect"
import { ChildProcess, ChildProcessSpawner } from "effect/process"

const version = Effect.gen(function*() {
  const spawner = yield* ChildProcessSpawner.ChildProcessSpawner
  return yield* spawner.string(ChildProcess.make("node", ["--version"]))
}).pipe(Effect.provide(NodeServices.layer))
```

| Operation | Use |
|-----------|-----|
| `spawner.string(command)` | Collect complete text output |
| `spawner.lines(command)` | Collect line-oriented output |
| `spawner.spawn(command)` | Scoped process handle and incremental streams |
| `ChildProcess.pipeTo(next)` | Compose command pipelines |

Use argument arrays for arguments. A string containing shell syntax is not a substitute for a structured command. Set environment overrides through command options, with `extendEnv` when the command should inherit the host environment.

Do not collect unbounded process output as a string. For long-lived tools or large outputs, consume the handle's streams as the process runs.

## Streaming and exit codes

```ts
import { NodeServices } from "@effect/platform-node"
import { Console, Effect, Schema, Stream } from "effect"
import { ChildProcess, ChildProcessSpawner } from "effect/process"

class CommandFailed extends Schema.TaggedError<CommandFailed>()("CommandFailed", {
  code: Schema.Number
}) {}

const lint = Effect.scoped(Effect.gen(function*() {
  const spawner = yield* ChildProcessSpawner.ChildProcessSpawner
  const handle = yield* spawner.spawn(ChildProcess.make("pnpm", ["lint"], {
    env: { FORCE_COLOR: "0" },
    extendEnv: true
  }))

  yield* handle.all.pipe(
    Stream.decodeText(),
    Stream.splitLines,
    Stream.runForEach((line) => Console.log(line))
  )

  const code = yield* handle.exitCode
  if (code !== ChildProcessSpawner.ExitCode(0)) {
    return yield* new CommandFailed({ code })
  }
})).pipe(Effect.provide(NodeServices.layer))
```

`spawn` is scoped: cancellation or scope closure tears down the owned process. Read the exit code even if the output stream completed normally; normal EOF does not imply success. If stdout and stderr are separate, drain them concurrently or use the combined stream so one full pipe cannot block the process.

Pipelines can fail at more than one stage. Decide whether a downstream short read is expected, such as a `head` command ending early, and preserve the exit semantics appropriate to the application.
