+++
id = "P-036"
title = "Plugin APIs: commands, queries, events — cross-plugin registries"
status = "deferred"
created = "2026-11-09"
owner = "devin"
priority = "high"
labels = ["architecture-platform"]
epic = "platform"
+++

## Context

Plugins should access other plugins' exposed APIs — not just static
contributions. Three new interaction kinds: **commands** (write),
**queries** (read), **events** (pub/sub). Example given: a `tables`
plugin offers commands/queries; the patients plugin registers its
tables through it.

## Key insight: commands/queries ≈ Effect services

Effect services ALREADY are typed cross-plugin APIs: a plugin's
Layer can require another plugin's service through `requiredDeps` +
the `R` channel (`Layer.provideMerge` composition). What's missing is
a *convention* — today "API" is implicit in whatever a service
exposes. Proposal: formalize `commands`/`queries` contribution kinds
that are sugar over Effect services, OR document that services ARE
the command/query surface and only add `events` as a new kind.

## Events — genuinely new

No pub/sub exists today. Proposal: an `EventsPlugin` kind +
`PlatformEvents` Effect service (typed topic map). Plugins publish
(`patient.created`) and subscribe (handler layers) without direct
service deps — decoupled cross-plugin reactions.

## The tables example — constraint

Drizzle schema is STATIC at build time; a runtime `tables.add`
command cannot produce drizzle tables post-hoc. Viable version:

- Table defs remain declarative contributions (as today).
- A `tables` plugin exposes a **registry query** (`tables.list`,
  `tables.get`) so other plugins introspect registered tables.
- app-host builds the drizzle schema FROM the registry — single
  assembly point instead of `Table` field merging.
- TRUE dynamic tables (runtime-created) are out of scope — would
  require abandoning drizzle's static schema.

## Open questions

- Are `commands`/`queries` just documented Effect services, or do
  they need their own contribution kind (uniform naming, discovery,
  dashboard introspection)?
- Event delivery: in-process sync only, or persisted/outbox for
  Cloudflare Durable Objects later?
- Should the dashboard plugin introspect commands/queries/events
  like it does contribution kinds today?
- Topic typing: string topics + Schema-validated payloads?

## Non-goals

- No runtime plugin loading — still static composition.
- No dynamic drizzle schemas.
