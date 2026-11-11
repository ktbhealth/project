+++
id = "P-046"
title = "Review adjustments for P-045: facade impls, dup-registration, services field, dep ids"
status = "deferred"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Review of P-045 (api plugins + service capabilities) surfaced five
adjustments. Base design stands; these are corrections to apply
during implementation.

## Adjustments

### 1. Facade methods bypass the bus — single impl source

Service methods must NOT round-trip through `queries.call` (which
pollutes error types with `| QueryNotFound`). The registered handler
and the service method are the SAME function:

```ts
make: Effect.gen(function* () {
  const queries = yield* Queries;
  const listImpl = Effect.fn("Tables.list")(function* () { ... });
  yield* queries.register(TablesApi.list, listImpl);
  return Tables.of({ list: listImpl, ... });
})
```

Bus is for discovery/dynamic callers; typed consumers get clean
`Effect<O, E>` signatures.

### 2. No spec/service drift

`api.ts` spec and service method describe the same operation —
they MUST share the impl function (adjustment 1 enforces this).
Spec = wire contract for dynamic callers; service method = typed
facade. Same handler underneath.

### 3. Duplicate registration = typed failure

`register(spec.id)` twice (double layer build, accidental
re-provide) must fail with a typed `DuplicateRegistration` error —
not silent last-wins. Surfaces composition bugs early.

### 4. `services` field accepts ServiceClass

Kernel tweak (~10 lines): `services` accepts `Layer | ServiceClass`
where the class has `.make`; app-host derives
`Layer.effect(Tag, Tag.make)`. Keeps plugin objects clean:
`services: Tables` instead of `services: Layer.effect(Tables, Tables.make)`.

### 5. Dep ids point at the registering plugin

Consumers calling `queries.call(PatientsApi.get, ...)` declare
`optionalDeps: ["patients.services"]` — the plugin object that
registers the specs — not a package-level id. `PluginRegistry`
already types this correctly.

## Status

Apply during implementation of P-045. Subagent building
api-query/api-command/api-event registry plugins now — adjustments
3 (dup registration) applies there.
