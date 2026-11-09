+++
id = "P-044"
title = "API plugins as buses: plugins expose/consume apis via bus registration in service layers"
status = "deferred"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-043 (and the whole contract/implements/uses chain).
Agreed design after review:

- API plugins ARE plugins — `api.query`, `api.command`, `api.event`
  provide bus services (QueryBus, CommandBus, EventBus).
- Plugins do NOT declare queries/commands in `definePlugin` — that
  was the wrong integration surface.
- Plugins EXPOSE apis by registering handlers on the buses inside
  their Effect service layer at build time.
- Plugins CONSUME apis by `bus.call(OtherPlugin.Api.method, input)`.
- Type-safety comes from exported signature VALUES in the plugin
  package (Schema in/out), not plugin fields.
- Plugin objects stay clean: id + deps + services (+ existing
  tables/routes/nav/panel/landing contributions).

## Design

### API plugins — the mechanisms

```ts
export const queryPlugin   = definePlugin({ id: "api.query",   services: QueryBusLive });
export const commandPlugin = definePlugin({ id: "api.command", services: CommandBusLive });
export const eventPlugin   = definePlugin({ id: "api.event",   services: EventBusLive });
```

Each bus is an Effect service: `register(name, signature, handler)`
/ `call(name, input)` / `publish(topic, payload)` / `subscribe(...)`.

### Tech plugin — tables

```ts
export const tablesPlugin = definePlugin({
  id: "tables",
  requiredDeps: ["api.query", "api.command"],
  services: TablesLive,
});
```

```ts
// TablesLive — registers handlers at layer build
const queries = yield* QueryBus;
const commands = yield* CommandBus;
yield* queries.register("tables.list", TablesApi.list, handler);
yield* commands.register("tables.register", TablesApi.register, handler);
```

### Domain plugin — patients (consumes + exposes, same mechanism)

```ts
export const patientsPlugin = definePlugin({
  id: "patients",
  requiredDeps: ["api.query", "api.command"],
  optionalDeps: ["tables"],
  services: PatientsLive,
});
```

```ts
// consume: bus.call with the other plugin's exported signature
const bus = yield* QueryBus;
const tables = yield* bus.call(TablesApi.list, undefined);  // Effect<TableInfo[]>
```

### Contracts — plain exported values, NOT plugin fields

```ts
// plugins/tables/src/api.ts
export const TablesApi = {
  list:     { input: Schema.Void, output: Schema.Array(TableInfo) },
  get:      { input: Schema.String, output: TableInfo },
  register: { input: RegisterTable, output: Schema.Void },
} as const;
```

Consumers import `TablesApi` for the signature (types + runtime
validation); the bus dispatches by name to whatever plugin
registered it.

## Type-safety story

- `bus.call(TablesApi.list, input)` — input/output types AND Schema
  validation derive from the signature value.
- Registration is checked: handler must match signature type.
- Plugin deps stay `requiredDeps`/`optionalDeps` on plugin ids —
  "uses tables" = `optionalDeps: ["tables"]` + import `TablesApi`.
- Missing provider at runtime → bus.call fails with typed
  "no handler" error; optionalDeps keeps it soft at compose time.

## Work items

- Kernel/infra: `QueryBusLive`, `CommandBusLive`, `EventBusLive`
  Effect services (in a shared package — `@ktb/api` or kernel).
- Bus semantics: registration conflicts (same name twice) → error;
  call to unregistered name → typed failure; events =
  publish/subscribe with Schema-validated payloads.
- Convention docs: `src/api.ts` per plugin exporting signature
  values; handlers registered in the plugin's Layer.
- Dashboard can introspect registered query/command/event names
  from the buses (live registry!) — nice bonus.

## Open questions

- Where do api plugins + bus services live: `plugins/api-*`
  packages vs `packages/api` infra? (Plugins dir keeps the "all is
  plugins" rule; packages/ keeps them out of the plugin registry.)
- Bus call scoping: can any plugin call any registered api, or
  should calls be scoped by deps (call allowed only if dep declared)?
- Event subscription: handler registered via EventBus in layer —
  lifecycle (unsubscribe on layer teardown)?
- Should `bus.call` be usable from React components (client-side)
  via server-fn bridge, or server-only v1?

## Non-goals

- No runtime plugin loading; no network/distributed buses; no new
  fields on Plugin for apis; services remain the impl vehicle.
