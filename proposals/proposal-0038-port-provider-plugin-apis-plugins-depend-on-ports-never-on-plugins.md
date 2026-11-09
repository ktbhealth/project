+++
id = "P-038"
title = "Port/provider plugin APIs: plugins depend on ports, never on plugins"
status = "deferred"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-037. Review found the contract-as-plugin design still
couples consumers to concrete provider packages (`uses: [tablesApi]`
imports `@ktb/tables` — package dep on the provider, single impl,
contract+impl versioned together). Requirement: TOTAL composability —
plugins depend on ports, implementations are swappable plugins.

## Design: port/provider split (Effect Tag pattern at plugin grain)

### Ports — stable contracts, no implementation

```ts
// @ktb/kernel or @ktb/ports — importable by any plugin, zero impl
export const TablesApi = definePort({
  queries:  { list: { input: Schema.Void, output: Schema.Array(TableInfo) } },
  commands: { register: { input: RegisterTable, output: Schema.Void } },
  events:   { "table.registered": TableRegistered },
});
// value carries ApiClient<T> phantom type — the dependency token
```

### Providers — plugins implement part or all of a port

```ts
// plugins/tables-d1
export const plugin = definePlugin({
  id: "tables.d1",
  provides: {
    [TablesApi]: {
      queries:  { list: ..., },
      commands: { register: ... },
      events:   { "table.registered": ... },
    },
  },
});
```

Multiple providers of the same port MERGE (host combines like
Layer.provideMerge) — e.g. `plugins/audit` provides only an event
handler for `table.registered`.

### Consumers — depend on the port value

```ts
// plugins/patients
export const plugin = definePlugin({
  id: "patients",
  uses: [TablesApi],        // port identity, NOT a plugin id
  services: PatientsLive,   // R channel gains typed TablesApi tag
});
```

## Machinery

- `definePort` — Schema-typed signatures + phantom `ApiClient` type;
  produces a unique symbol-keyed token (port identity).
- `Plugin` union gains `provides` contribution kind; `uses` joins
  `requiredDeps`/`optionalDeps` as a dep edge keyed by port token.
- app-host builds `PluginApis` layer: merge all `provides` per port,
  expose one Tag per port. `uses` adds the Tag to consumer R channel.
- Missing provider + `uses` = compile error; `usesOptional` =
  runtime-checked skip.
- Events: port declares typed topics; any provider publishes, any
  consumer subscribes — naturally many-to-many.

## What this buys (vs P-037)

- Swap `tables.d1` ↔ `tables.memory` with zero consumer changes
- Partial providers compose (audit/logging plugins add handlers)
- Consumers never depend on provider packages — only on `@ktb/kernel`
  ports
- Ports version independently of implementations

## Open questions

- Where do ports live: `@ktb/kernel` (all ports in one pkg) vs
  `@ktb/ports` per-domain packages vs ports contributed by infra
  plugins themselves (e.g. `plugins/tables` exports the TablesApi
  port AND a default provider — consumers import port from it but
  don't depend on the impl being present)?
- Merge semantics for conflicting providers (two impls of same
  query) — first-wins, error, or explicit priority field?
- `uses` vs `requiredDeps`: does `uses: [Port]` also imply an
  ordering edge? (Probably yes — provider must build first.)
- Client-side (browser) access to ports — queries from React
  components need a fetch/RPC bridge or is api in-process only?

## Non-goals

- No runtime plugin loading; no network/distributed APIs; no
dynamic drizzle schemas (tables registry remains a query over
static defs).
