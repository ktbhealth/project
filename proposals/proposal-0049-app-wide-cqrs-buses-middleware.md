+++
id = "P-049"
title = "App-wide CQRS buses + middleware"
status = "in-progress"
created = "2026-11-09"
priority = "low"
+++

## Context

Plugin-plugin communication runs on @effect-server-utils/cqrs typed surfaces: each plugin registers `Query.handlersOf`/`Command.handlersOf` layers and consumers use `Query.dispatcher(Group)`/`Command.dispatcher(Group)` with compile-time `Registered<>` enforcement. This covers every current need.

## Scope

Two deferred capabilities, both centered on the composition root (packages/app-host):

### 1. App-wide QueryBus / CommandBus dispatch tables

`QueryBus.execute(Def, payload)` / `CommandBus.execute(Def, payload)` dispatch by message definition without a static group reference — needed only for:

- dynamic dispatch (message defs chosen at runtime, e.g. admin tooling, generic CRUD UIs)
- optional integrations where the consumer can't declare a compile-time dep on the provider's group

Requires a convention for enumerating registered groups at composition time (the host can't see groups today — each plugin's services layer just merges handlersOf outputs). Options: plugins export their groups for host collection, a registry plugin kind, or `Registered<>` tag introspection. Deferred until a real dynamic-dispatch consumer exists — typed dispatchers are strictly better where applicable.

### 2. Bus middleware

The library supports middleware pipelines (authz, logging, tracing, metrics) at the dispatch boundary. Candidate uses:

- authz: enforce `requireCanAs`-style checks uniformly for cross-plugin calls (today each handler self-authzs — inconsistent)
- observability: span attributes per dispatch (the lib already emits spans; middleware adds domain attributes like ownerId/userId)
- audit: log every cross-plugin command for the platform console

## Triggers to revisit

- A plugin needs to call an API it can't statically depend on (optional feature discovery)
- Cross-plugin authz drift becomes a real bug class
- Need unified audit/metrics over plugin calls

## Status

Deferred — no current consumer. Typed dispatchers + per-handler authz cover all present use cases.
