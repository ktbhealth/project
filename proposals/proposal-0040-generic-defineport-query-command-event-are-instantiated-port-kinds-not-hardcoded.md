+++
id = "P-040"
title = "Generic definePort: query/command/event are instantiated port kinds, not hardcoded"
status = "deferred"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-039. Review: query/command/event should not be
hardcoded kernel kinds — the port itself is the generic primitive;
q/c/e are just SHAPES you instantiate. New interaction styles
(subscriptions, caches, flags) should need zero kernel changes.

## Design: one primitive, kind as metadata

```ts
// @ktb/kernel — the ONLY primitive
export function definePort<S extends PortShape>(port: S): Port<S>

// instances — kind is advisory metadata, not machinery
TablesQueries  = definePort({ kind: "query",   methods: {...} })
TablesCommands = definePort({ kind: "command", methods: {...} })
TablesEvents   = definePort({ kind: "event",   topics: {...} })
SessionPort    = definePort({ kind: "session", methods: {...} })  // custom kind OK
```

PortShape variants: `methods` (request/response — queries, commands,
sessions) vs `topics` (pub/sub — events). `kind` is a string tag used
for dashboard grouping and future per-kind semantics (command
idempotency, event delivery guarantees) — the provider/consumer
machinery is identical for all.

## Machinery (kind-agnostic)

- `Plugin` union gains ONE new member: `ProvidesPlugin`
  (`provides: { [Port]: impl }`) — no per-kind union members needed.
- `uses`/`usesOptional` dep edges keyed by port token, kind-agnostic.
- app-host: merge all `provides` per port token → one Tag per port
  in a `PluginPorts` layer; consumers get typed `ApiClient` in R
  channel.
- Type-safety: port value carries phantom client type; Schema
  signatures validate at boundary; `uses` checked against port
  registry.

## What this buys

- New interaction kinds = new `definePort` instances, zero kernel
  work
- q/c/e remain first-class via `kind` metadata (dashboard groups,
  per-kind semantics later)
- Consumers/providers never know or care about kinds — only port
  identity + types

## Open questions

- PortShape: `methods` vs `topics` — two shapes or one unified
  (topics = methods with no input/return)? Unified is simpler;
  separate gives events different delivery semantics naturally.
- Port location: `@ktb/kernel` vs `@ktb/ports` packages — ports are
  contracts, arguably belong in domain-adjacent packages.
- Per-kind semantics hooks: does `kind: "command"` unlock
  idempotency/audit middleware automatically, or is that opt-in per
  port?
- Client-side (browser) port access — RPC bridge or in-process only?

## Non-goals

- No runtime plugin loading; no network/distributed ports; no
dynamic schemas; kind semantics beyond metadata in v1.
