+++
id = "P-043"
title = "All-plugins model: contract/implements/uses roles — no separate port concept"
status = "deferred"
created = "2026-11-09"
owner = "devin"
priority = "high"
labels = ["architecture-platform"]
epic = "platform"
+++

## Context

Supersedes P-042. Final simplification: DROP the "port" concept —
everything is a plugin. A contract IS a plugin that declares
signatures; an implementation IS a plugin that satisfies it; a
consumer IS a plugin that needs it. Three roles, one concept, all
registry-tracked by id.

## Design

### Contract plugins — signatures only

```ts
export const PatientReads = definePlugin({
  id: "patients.reads",
  contract: {
    list: { input: PatientFilters, output: Schema.Array(PatientSchema) },
    get:  { input: IdSchema, output: PatientSchema },
  },
});
```

### Contract composition — a contract declares what it merges

```ts
export const PatientsApi = definePlugin({
  id: "patients.api",
  uses: [PatientReads, PatientWrites],   // what it needs to compose
  contract: composeContracts(PatientReads, PatientWrites),
});
```

### Impl plugins — satisfy a contract, independently

```ts
export const patientsQueriesPlugin = definePlugin({
  id: "patients.queries",
  implements: PatientReads,     // impl type-checked vs contract
  services: QueriesLive,        // impl via Effect layer as today
});
```

### Consumer plugins — need a contract, get typed client

```ts
export const patientsPanelPlugin = definePlugin({
  id: "patients.panel",
  uses: [PatientReads],         // typed dep on contract plugin
  panel: { sidebar: [...] },
});
```

## Contract changes

- `Plugin` union gains `ContractPlugin` (`contract` field) — the
  only new contribution kind; `implements`/`uses` are dep-edge
  fields on PluginBase alongside requiredDeps/optionalDeps.
- `uses` entries must resolve to ContractPlugins; `implements`
  entries likewise. Registry/typecheck enforces.
- `composeContracts(...)` merges signature maps; name conflicts =
  type error.
- app-host: for each contract, merge all `implements` impls → one
  typed Tag; consumers' `uses` adds Tag to R channel with
  contract-derived client type.

## Type-safety

- Contract plugin carries phantom `ApiClient` type → consumers get
  `list(): Effect<Patient[]>` inferred.
- `implements` impl is checked against contract signatures.
- `uses`/`implements`/`contract-composition` all registry-checked.

## What this unifies

- Queries, commands, events = just contract plugins (topics =
  contract shape variant) — no special kinds.
- Services remain internal domain logic; contracts are the public
  surface.
- Discovery/ordering/registry unchanged — contracts are plugins.

## Open questions

- `uses` on a contract plugin (composition) vs `uses` on impl/
  consumer plugins — same field, different role semantics? Probably
  fine: contracts compose contracts, impls/consumers use contracts.
- Contract shape variants: methods-map vs topics-map — one
  `contract` field handling both or `contract` + `eventContract`?
- Multiple impls of same contract — merge (Layer semantics) or
  error on conflict?
- Where contract plugins live: same package as domain (exports from
  `@ktb/patients`) — consumers import contract, not impl.

## Non-goals

- No runtime plugin loading; no network contracts; no dynamic
  schemas; services stay the impl vehicle (no new runtime kind).
