+++
id = "P-039"
title = "Independent query/command/event plugin kinds — per-kind ports and providers"
status = "open"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-038. Review: bundling queries+commands+events into one
`api`/`provides` object is still not fully composable — a read-only
consumer shouldn't depend on the write side. Requirement: query,
command, and event are INDEPENDENT plugin kinds, each with its own
port, providers, and consumers.

## Design

### Three port families (separate contract values)

```ts
TablesQueries  = defineQueryPort({ list: {...}, get: {...} })
TablesCommands = defineCommandPort({ register: {...} })
TablesEvents   = defineEventPort({ "table.registered": Payload })
```

Each port = Schema-typed signatures + phantom client type + unique
identity token. Lives in kernel/ports — importable by anyone, zero
impl.

### Providers — one plugin object per kind

```ts
{ id: "tables.queries",  provides: { [TablesQueries]: {...} } }
{ id: "tables.commands", provides: { [TablesCommands]: {...} } }
{ id: "tables.events",   provides: { [TablesEvents]: {...} } }
```

Install/swap/omit each independently. Multiple providers of the same
port merge (Layer semantics).

### Consumers — per-kind `uses`

```ts
{ id: "patients", uses: [TablesQueries], services: PatientsLive }
{ id: "importer", uses: [TablesCommands, TablesEvents] }
```

`uses` entries are port tokens → typed Tags in the consumer's R
channel. Missing provider = compile error (`uses`) or runtime-checked
skip (`usesOptional`).

## Contract changes

- `Plugin` union gains `QueryPlugin`, `CommandPlugin`, `EventPlugin`
  (or a single `ProvidesPlugin` parameterized by port kind — decide
  in implementation; union members preferred for ForbiddenContributions
  consistency).
- `uses`/`usesOptional` join `requiredDeps`/`optionalDeps`.
- app-host builds one Tag per provided port; merges multi-providers.
- Events: port declares typed topics; publishers AND subscribers both
  `uses` the event port — many-to-many by construction.

## Type-safety

- Ports carry `ApiClient` phantom types → `TablesQueries.list()`
  returns `Effect<Output, Error>` inferred.
- Schema validation at the port boundary (runtime) + TS (compile).
- `uses` checked against port registry — unknown port = type error.

## Open questions

- Do queries/commands/events need separate `defineQueryPort`/
  `defineCommandPort`/`defineEventPort`, or one `definePort` with a
  kind tag? (Separate = clearer union kinds; one = less machinery.)
- Query vs command distinction: is it just naming (read vs write
  intent) or do commands get different semantics (idempotency keys,
  auditing)?
- Events: should subscribe handlers be part of `provides` or a
  separate `subscribes` contribution?
- Port location: `@ktb/kernel` vs `@ktb/ports` packages.
- Client-side access (React components calling queries) — deferred
  or needs an RPC bridge?

## Non-goals

- No runtime plugin loading; no network APIs; no dynamic drizzle
  schemas; no cross-process events (in-process runtime only).
