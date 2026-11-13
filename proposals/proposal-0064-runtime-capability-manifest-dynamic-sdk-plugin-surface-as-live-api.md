+++
id = "P-064"
title = "Runtime capability manifest + dynamic SDK: plugin surface as live API"
status = "open"
created = "2026-11-12"
owner = "devin"
priority = "med"
labels = ["architecture-platform", "plugins"]
links = ["P-047", "P-058"]
epic = "platform"
+++

## Context

The CQRS layer (@effect-server-utils/cqrs, P-047) already gives every plugin a typed cross-plugin surface: `plugins/*/src/api.ts` declares Query/Command/Event defs with effect/Schema payload/success/failure. Today that surface is consumed in-process only. The obvious next step — a public REST API + generated SDK — fails as a *static* artifact: the plugin set is runtime-mutable (P-058 enable/disable/config in /platform/plugins), so a checked-in OpenAPI spec would misrepresent the live surface the moment a plugin is toggled.

Note: plugins are bundled at build time today (discovery via import.meta.glob); enablement is runtime. The manifest must reflect the *enabled* set, not the bundled set.

## Proposal

1. **Runtime capability manifest endpoint** — e.g. `GET /api/platform/capabilities` (platform plugin surface). At request time, for every ENABLED plugin, emit its declared Query/Command/Event groups with payloads/success/failure serialized via `Schema.toJsonSchemaDocument` (same mechanism as plugin config forms, P-058). Disabled plugins contribute nothing — the manifest IS the truth, zero drift.
2. **Dynamic dispatch endpoint** — thin transport over the existing buses: `POST /api/cqrs/:group/:tag` → `QueryBus.execute`/`CommandBus.execute`, gated by the same pluginGate middleware the buses already use. Events via `EventBus.stream` → SSE for subscriptions.
3. **Dynamic SDK** — small fetch client (`createClient({ baseUrl, credentials })`) exposing `query(tag, payload)`, `command(tag, payload)`, `on(event, handler)`. Payloads validated client-side against the fetched manifest (JSON-schema check), not static types.
4. **Optional static snapshot** — per-deployment `ktb codegen <env>` can fetch the live manifest and emit TS types/OpenAPI for teams that want compile-time types; derived artifact, never source of truth.

## Open questions

- **AuthZ scoping**: CQRS dispatch today runs under plugin-internal trust. HTTP transport needs external credential → Principal mapping and per-call `runAs`-equivalent scoping — real work, likely reuses @ktb/auth session + @ktb/authz.
- **Surface curation**: not every CQRS def should be externally callable (admin/internal queries). Need an `expose: true` marker on defs or group-level opt-in.
- **Rate limiting / abuse**: public endpoint dispatching arbitrary commands needs throttling.
- Manifest size: fine at current plugin count; may need per-plugin lazy fetch later.

## Work items (rough)

- Spike: manifest endpoint for ONE plugin (patients) — defs → JSON schema → response shape.
- Dispatch endpoint + authZ scoping spike.
- Dynamic client package (`packages/sdk` or `@ktb/sdk`) with manifest fetch + validation.
- `expose` opt-in convention on api.ts defs.
- Docs + example consumer.
