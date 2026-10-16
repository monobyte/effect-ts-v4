# Effect v4 Agent Skill

Curated, self-contained Effect v4 guidance for AI coding agents.

## Install

```sh
npx skills add monobyte/effect-ts-v4
```

The skill keeps a small entry point and topic guides that agents can read as needed. Each guide explains the relevant choices, current API conventions, common mistakes and focused examples.

Coverage includes core Effect, services/layers, data types, concurrency, streaming, Schema, transactions, application patterns, migration and ecosystem wiring. Focused guides cover HTTP/SQL, AI, batching/GraphQL, testing/observability and CLI/child processes.

Reference material is embedded as Markdown. No library source, example source tree or generated API catalogue is bundled.

The guides were checked against [Effect's AI documentation](https://github.com/Effect-TS/effect/tree/main/ai-docs/src) and current public APIs at commit `4d003f90ca39caca54fb0c495cf82e6a159d63f0`. Check the application's installed versions when using individual APIs. Adapted upstream examples retain the [upstream licence](effect-ts-v4/docs/LICENSE.effect).
