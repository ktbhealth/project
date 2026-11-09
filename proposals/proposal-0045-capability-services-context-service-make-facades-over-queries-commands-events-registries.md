+++
id = "P-045"
title = "Capability services: Context.Service.make facades over Queries/Commands/Events registries"
status = "deferred"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-044. Final design after brainstorming: api plugins
provide registry services (transport + discovery); domain/tech
plugins expose CAPABILITIES — typed facades built with the
Effect-v4 `Context.Service` `make` convention. Two consumption
paths: typed capability (hard/soft dep) or dynamic registry call.

## Architecture

### Layer 1 — api plugins (transport + discovery)

`plugins/api-query`, `plugins/api-command`, `plugins/api-event` —
each a plugin providing a registry service built via
`Context.Service(...)("id", { make })`:

- `Queries`: `register(spec, handler)` / `call(spec, input)` —
  scoped registration (auto-unregister on layer teardown)
- `Commands`: `register(spec, handler)` / `execute(spec, input)`
- `Events`: `publish(spec, payload)` / `subscribe(spec, handler)` /
  `stream(spec)`

Specs are plain literals `satisfies QuerySpec/CommandSpec/EventSpec`
— `{id, input: Schema, output: Schema, error: Schema}` — exported
from the plugin's `api.ts`. No `define*` helpers; `satisfies`
preserves concrete types for inference. Spec constructors live in
the api plugins, NOT kernel.

### Layer 2 — capabilities (typed facades)

A plugin's public API is a `Context.Service` class whose `make`
registers its specs on the buses AND returns a typed facade:

```ts
export class TablesCapability extends Context.Service<...>()(
  "@ktb/tables/TablesCapability",
  { make: Effect.gen(function*() {
      const queries = yield* Queries; ...
      yield* queries.register(TablesApi.list, ...);
      return TablesCapability.of({ listTables: ..., ... });
  })}
) {}
```

### Layer 3 — consumers, two paths

```ts
// typed capability — hard dep (R-enforced) or soft (serviceOption)
const tables = yield* TablesCapability;
const maybe = yield* Effect.serviceOption(TablesCapability);

// dynamic — registry call with spec token (no static dep)
const q = yield* Queries;
q.call(TablesApi.list, undefined)   // needs optionalDeps: ["tables"]
```

## Plugin examples (agreed shape)

- `plugins/tables` (tech): `TablesCapability` + `TablesApi` specs;
  `requiredDeps: ["api.query","api.command","api.event"]`
- `plugins/patients` (domain): `requiredDeps: ["api.query"]`,
  `optionalDeps: ["tables"]`; consumes via `TablesCapability` or
  `queries.call(PatientsApi.*)`; exposes `PatientsCapability`

## Type-safety

- Spec literals carry Schema types → `call`/`execute` infer
  I/O/E per spec; boundary validated by Schema
- Capability methods are plain typed Effects — R-channel deps
- `serviceOption` for soft capability deps; `QueryNotFound` typed
  error for missing dynamic registrations
- Scoped registration → plugin teardown removes its apis

## Open questions

- `services` field: accept `ServiceClass` with `.make` directly
  (host derives `Layer.effect(Tag, Tag.make)`)? Recommended.
- Capability vs raw registry: convention that domain code uses
  capabilities, registry only for infra/dynamic cases?
- Events capability shape: `onX(): Stream` vs `subscribe(handler)`?
- Where api plugins live: `plugins/api-*` (all-plugins rule) —
  confirm.

## Non-goals

- No runtime plugin loading; no network buses; kernel stays
  transport-agnostic (api plugins own spec types).
