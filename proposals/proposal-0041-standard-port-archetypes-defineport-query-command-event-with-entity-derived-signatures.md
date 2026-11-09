+++
id = "P-041"
title = "Standard port archetypes: definePort.query/command/event with entity-derived signatures"
status = "open"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-040. Review: a fully generic `definePort` pushes
shape-design onto every plugin — consumers can't assume a Query port
offers `list`/`get`, so type-safety isn't natural. Requirement:
kernel defines what each port KIND offers; plugins supply entity
Schema + optional extras; signatures are DERIVED, not hand-written.

## Design: archetypes parameterized by Schema

```ts
// @ktb/kernel
export const PatientsQueries = definePort.query({
  entity: PatientSchema,          // drives all method types
  filters: PatientFiltersSchema,  // list() input
  methods: {                      // optional domain extras
    searchByName: { input: Schema.String, output: Schema.Array(PatientSchema) },
  },
});
// derived client:
//   list(filters?): Effect<Patient[]>
//   get(id): Effect<Patient>
//   searchByName(name): Effect<Patient[]>

export const PatientsCommands = definePort.command({
  entity: PatientSchema,          // → create/update/delete derived
  methods: { archive: {...} },    // extra verbs OK
});

export const PatientsEvents = definePort.event({
  "patient.created": PatientSchema,   // typed topics
  "patient.archived": IdSchema,
});
```

## Why this is naturally type-safe

- Archetype OWNS the standard signatures (`list`, `get`, `create`,
  `update`, `delete`) — plugins cannot omit or mistype them.
- Entity Schema is the single source → input/output/error types all
  derived; wire validation and TS types can't drift.
- `filters`/`methods` are strictly additive — extensions are
  type-checked against the same Schema machinery.

## Composability payoff

Every `query` port exposes the same `list`/`get` shape → generic
consumers work across domains:

```ts
function AdminTable<Q extends QueryPortClient<any, any>>({ port }: { port: Q })
// PatientsQueries / ConsultationsQueries / UsersQueries — all fit
```

One admin-table component, one data hook, one set of tests — new
domain ports drop in without UI changes.

## Machinery (unchanged from P-040)

- `provides: { [Port]: impl }` / `uses: [Port]` — port tokens,
  kind-agnostic at the mechanism level (kind now structural via
  archetype).
- app-host merges providers → typed Tags; Schema validates boundary.

## Open questions

- Standard method set per archetype: is `list`/`get` enough for
  queries? Add `search`/`count`? Commands: `create`/`update`/
  `delete` sufficient, or `patch` semantics?
- Do archetypes enforce handler IMPLEMENTATION shape too (provider
  must implement exactly the derived methods)? Presumably yes —
  `provides` impl type = derived client type.
- Events archetype: topics-only, or also standard lifecycle topics
  (`*.created`/`*.updated`/`*.deleted` auto-derived from entity)?
- Where archetypes live: `@ktb/kernel` (with Schema dep) vs
  `@ktb/ports`.

## Non-goals

- No runtime plugin loading; no network ports; no dynamic schemas;
  archetypes cover query/command/event — other port kinds can still
  use raw `definePort` if needed.
