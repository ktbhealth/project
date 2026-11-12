+++
id = "P-047"
title = "Adopt @effect-server-utils/cqrs for plugin-plugin communication"
status = "done"
created = "2026-11-09"
owner = "devin"
priority = "high"
epic = "platform"
+++

## Context

Supersedes P-044/045/046 (custom registry design). Research found
`@effect-server-utils/cqrs` (dataquail, v0.1.0-beta.10, peer
effect@^4) — typed Command/Query/Event buses with a more sophisticated
design than our sketch. SPIKE VERIFIED end-to-end in this repo:
buses, typed dispatchers, handlersOf layer inference, event
subscribe/stream all work with effect@4.1.2.

## What the lib gives us (verified)

- `Query.make/Command.make(tag, {payload, success, failure})` —
  Schema-typed message definitions
- `X.group(...)` / `X.handlersOf(group, handlers)` → Layer with
  handler service requirements INFERRED into R channel
- `X.dispatcher(group)` → per-tag typed methods
  (`q["names.list"](payload)`), requires branded `Registered<G>`
- `QueryBus`/`CommandBus` app-wide buses via `mergeDispatchTables`
  at composition root; `bus.execute(Def, payload)` typed off def
- `EventBus`: `subscribe` (in-publisher-fiber, can roll back) /
  `subscribeAfterCommit` (isolated) / `stream` — consistency model
  chosen at subscription
- Middleware (span default; metrics/deadline opt-in), Saga,
  DeferralSink seam for future unit-of-work

## Integration model (plugins)

- Plugin `services` field = `X.handlersOf(group, handlers)` layer —
  zero new kernel machinery; deps flow through Layer R channels
- Each plugin exports `api.ts` with `Query.make`/`Command.make`/
  `Event.make` definitions + `X.group`s
- Consumers: typed `X.dispatcher(group)` (compile-enforced via
  Registered<>) OR `bus.execute` for dynamic cases
- app-host composition root: merge all dispatch tables →
  `makeQueryBus`/`makeCommandBus`; provide `makeEventBus()` layer
- Plugin dep edges still via requiredDeps/optionalDeps on plugin ids

## Risks / open questions

- BETA (0.1.0-beta.10, first publish Aug 2026, ~4k monthly dl) —
  API churn possible; pin exact version, evaluate at 1.0
- Our earlier concerns still apply: service capabilities for direct
  deps may still be nicer than bus dispatch for hard deps — decide
  per case (bus for cross-plugin, services for internal)
- `services: ServiceClass` kernel tweak (P-046 adj.4) still useful
  but independent of this choice
- Events: subscribeAfterCommit needs DeferralSink/unit-of-work to
  mean "after commit" — without it runs at end of dispatch (fine v1)

## Work items

- Move dep from root package.json to the packages that use it
- Spike file plugins/users/spike-cqrs.ts was the verification —
  delete after integration pattern is proven in one real plugin pair
- First real integration: pick smallest pair (users ↔ groups or
  patients ↔ consultations) and convert one query + one command
- app-host: wire dispatch-table merge + bus layers into createApp
