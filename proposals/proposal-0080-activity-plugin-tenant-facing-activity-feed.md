+++
id = "P-080"
title = "Audit feed: permission-scoped activity view — decouple audit from PlatformAdmin"
status = "open"
created = "2026-11-15"
priority = "med"
+++

## Problem

The audit pipeline already exists — `admin_audit` table + `Audit.record`
port, with emitters in `groups`, `patients`, `rbac`, `users`,
`consultations` — but it is admin-only end to end: the sole read path is
`/platform/audit` behind `PlatformAdmin`, and emission is limited to
admin-gated mutations by convention. Staff have no "what changed
recently" view, and day-to-day domain events (note added, transcription
finished, appointment changed) are never recorded.

## Proposal

Reuse the existing table and port; decouple the read side from admin and
widen emission. No new plugin, no new table.

- **Audit stays an optional plugin** — `audit.services` is consumed via
  `optionalDeps` (the pattern `groups` already uses for
  `rbacServicesPlugin`/`auditServicesPlugin`). Domain plugins must keep
  working when audit is absent: the port resolves to a no-op (or
  `Option`) at composition so `Audit.record` calls don't break writes.
- **Permission-scoped read** — add a scoped list path on the audit
  service: tenant filter from the bound `Principal`/`@ktb/tenant`,
  guarded by an RBAC permission (e.g. `audit.read`) via `@ktb/authz`
  (`Forbidden` on denial). `PlatformAdmin` keeps the global view.
- **Staff-facing page** — new panel page `/activity`: paginated feed
  (actor, action, target, tenant, time), day grouping, entity filter,
  deep links into source records where resolvable. Platform-global rows
  (`tenant_id NULL`) stay admin-only.
- **Sidebar** — new `Governance` group in `console.sidebar`; Audit log
  moves out of `System` into it, alongside the new Activity entry.
- **Emission widening** — non-admin services start recording domain
  events through `Audit.record`: consultations (note added, status
  change), transcription (completed/failed), patients. `detail` stays
  ids/flags only — never PHI, transcripts, or clinical content.

## Boundaries

- Same `admin_audit` table, same `Audit.record` port — no parallel
  `activity_events` store. A separate event stream only earns its keep
  if payloads outgrow the `detail` JSON cap (4000 chars) or need
  richer typing.
- The compliance properties don't change: append-only, actor
  denormalized off Principal, tenant-scoped.
- Positioning rule holds: events are factual ("note added"), never
  interpretive.

## Open questions

- Permission granularity: single tenant-scoped `audit.read` vs
  per-domain permissions. Proposal assumes `audit.read` for v1.
- Feed composition: show all tenant events, or let roles filter by
  entity type? v1 shows all tenant-scoped rows; per-type filtering is
  a UI concern.
- Naming: `/activity` for staff vs keeping everything under
  `/platform/*` with a non-admin guard — lean `/activity` since it's a
  workspace surface, not platform admin.
