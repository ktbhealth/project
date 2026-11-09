+++
id = "P-042"
title = "Composable port fragments: definePort + compose, consumers/providers bind fragments"
status = "open"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-041 (archetypes = hardcoding moved one level down).
Correct model: a PORT is a composable contract fragment — like
interface composition. No query/command/event archetypes, no kinds.
You define small ports and `compose` them; consumers and providers
bind exactly the fragments they need.

## Design

### Port fragments — plain composable contracts

```ts
// plugins/patients/src/ports.ts
export const PatientReads = definePort({
  list: { input: PatientFilters, output: Schema.Array(PatientSchema) },
  get:  { input: IdSchema, output: PatientSchema },
});
export const PatientWrites = definePort({
  create:  { input: PatientCreate, output: PatientSchema },
  archive: { input: IdSchema, output: Schema.Void },
});
export const PatientEvents = definePort({
  "patient.created": PatientSchema,
  "patient.archived": IdSchema,
});

// composition — client type is the intersection, fully inferred
export const PatientsApi = definePort.compose(PatientReads, PatientWrites, PatientEvents);
```

- `definePort` takes a method map (or topic map for events) of
  Schema-typed signatures → produces a port value carrying a phantom
  `ApiClient` type + unique identity token.
- `compose(...)` merges fragments: method map union, topics union,
  client type = intersection. Conflicts (same name, different
  signature) = type error.

### Consumers bind fragments

```ts
{ id: "patients.panel",   uses: [PatientReads] }              // read-only
{ id: "importer",         uses: [PatientWrites, PatientEvents] }
{ id: "patients.services", uses: [PatientsApi] }              // composed
```

### Providers bind fragments — independently

```ts
{ id: "patients.queries",  provides: { [PatientReads]: {...} } }
{ id: "patients.commands", provides: { [PatientWrites]: {...} } }
{ id: "patients.events",   provides: { [PatientEvents]: {...} } }
```

Different plugins can provide different fragments of the same domain
surface; a composed port's providers need not come from one plugin.

## Machinery

- `Plugin` union: `ProvidesPlugin` member (`provides` keyed by port
  token); `uses`/`usesOptional` dep edges keyed by port token.
- app-host merges all `provides` per port token → one typed Tag per
  port in `PluginPorts` layer; consumers get `ApiClient` in R
  channel; Schema validates at boundary.
- Missing provider + `uses` = compile error; `usesOptional` =
  runtime-checked skip.
- "query"/"command"/"event" are naming conventions for fragments —
  NOT kernel kinds. Events are just ports whose shape is topics.

## Type-safety

- Port value carries phantom client type → `PatientReads.list()`
  returns `Effect<Patient[], Err>` inferred.
- `compose` conflicts are compile errors.
- `uses`/`provides` keyed by port identity — unknown port = type
  error; registry-checked like `requiredDeps`.

## Open questions

- Port shape: methods-map vs topics-map — one `definePort` handling
  both (topics = methods with payload-only signature) or two
  helpers (`definePort`, `defineEventPort`)?
- `compose` semantics for overlapping method names — error (strict)
  or override-with-priority?
- Port location: exported from domain plugin packages
  (`@ktb/patients/ports`) vs `@ktb/ports` — consumers import the
  FRAGMENT, never the provider plugin.
- Client-side access — in-process only v1?

## Non-goals

- No runtime plugin loading; no network ports; no dynamic schemas;
  no hardcoded interaction kinds.
