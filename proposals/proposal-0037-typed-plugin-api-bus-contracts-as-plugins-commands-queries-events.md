+++
id = "P-037"
title = "Typed plugin API bus: contracts as plugins — commands, queries, events"
status = "open"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-036. Requirements from review:

- queries/commands/events ARE plugins — they live in `plugins/` like
  every other contribution, not as anonymous fields.
- Plugins must be able to call each other's exposed APIs
  (plugin-plugin communication).
- Full type-safety, declarative — no stringly-typed lookups.

## Design: contract-first API plugins

Each API surface is a **contract value** (`defineApi`) pairing
Schema-typed signatures with an implementation, contributed by a
plugin object:

```ts
// plugins/tables
export const tablesApi = defineApi({
  queries:   { list: {input, output}, get: {...} },
  commands:  { register: {...} },
  events:    { "table.registered": PayloadSchema },
});

export const tablesPlugin = definePlugin({
  id: "tables",
  api: tablesApi.implement({ queries: {...}, commands: {...}, events: {...} }),
});
```

Consumers declare `uses` — a typed dep on the contract, not an id:

```ts
// plugins/patients
export const patientsPlugin = definePlugin({
  id: "patients",
  uses: [tablesApi],
  services: PatientsLive,   // TablesApi tag in R channel, fully typed
});
```

## Kernel/host machinery

- `defineApi` produces a contract object: schemas (runtime validation)
  + phantom types (compile-time). `implement()` returns the
  contribution payload.
- `Plugin` union gains `ApiPlugin` kind: `{ api: ImplementedApi }`.
- app-host builds a `PluginApis` Effect layer: one typed service Tag
  per api plugin. `uses` adds required Tags to the consumer layer's
  R channel; a missing api plugin fails typecheck (or is skipped for
  `usesOptional`).
- Events: contract declares typed topics; `implement` provides
  publishers, `uses` provides subscribers — delivery in-process via
  the runtime (persistence/outbox deferred).

## Type-safety story

- Consumer imports `tablesApi` → `TablesApi.queries.list()` returns
  `Effect<Output, Error>` — compile-time typed, schema-validated at
  the boundary.
- `uses: [tablesApi]` is checked against `PluginRegistry` like
  `requiredDeps` — unknown api = type error.
- Dashboard can introspect api plugins (id + query/command/event
  names) like other contribution kinds.

## Resolved questions from P-036

- Commands/queries get their own kinds via `api` — uniform,
  introspectable, decoupled from "services" (services stay internal
  domain logic; api is the public surface).
- Tables: registry query (`tablesApi.queries.list`) + schema built
  from registry at composition; NO runtime drizzle tables.

## Open questions

- `uses` vs reusing `requiredDeps` — `uses` carries the contract
  type; `requiredDeps` stays for ordering-only edges. Keep both?
- Should api plugins be allowed to `uses` other apis (chaining)?
- Error channel: schema-typed errors per method, or generic?
- Versioning: contract + impl in same package — breaking change =
  new api id (`tables.v2`) or semver on the package?

## Non-goals

- No runtime plugin loading, no dynamic drizzle schemas, no
  cross-process/network APIs (in-process only for now).
