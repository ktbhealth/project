+++
id = "P-080"
title = "Activity plugin: tenant-facing activity feed"
status = "open"
created = "2026-11-15"
priority = "med"
+++

## Problem

Staff have no place to see "what happened recently" — patient changes, consultation events, transcription runs, admin actions are scattered across per-domain screens. The existing `audit` plugin is an admin-only audit log under `/platform/audit`; there's no lightweight, read-mostly activity timeline for regular users.

## Proposal

New `plugins/activity` (`activity.*` concerns) contributing:

- **`activity.services`** — ActivityService behind a port: `record(event)`, `list({ scope, cursor })`. Events are domain facts (entity, action, actor, tenant, timestamp, refs) emitted by other plugins — not the audit plugin's compliance log (that stays admin-scoped).
- **`activity.tables`** — `activity_events` table (tenant-scoped, append-only, indexed by tenant+created_at).
- **`activity.panel`** — `/activity` page: paginated feed grouped by day, filter by entity type, deep links into source records.
- **`activity.chrome`** — nav entry (probably under an existing group in console sidebar).
- **Emitters**: consult `patients`, `consultations`, `transcription-*`, `users` for the highest-signal events in v1 (patient created/updated, consultation created, transcription completed/failed); other plugins can opt in later via the port.

## Boundaries

- Not a duplicate of `audit`: activity is operational recency ("what changed"), audit is compliance ("who did what, immutable, admin-only"). Activity events may reference audit records but don't carry the same retention/immutability guarantees.
- Architecture rules apply: ports/plugins can't touch adapters; emission happens via the service port from plugins, storage adapter lives in `packages/` if needed (or reuse existing db infra).
- Positioning rule: no clinical interpretation — events are factual ("note added"), never interpretive ("diagnosis suggested").

## Open questions

- Feed scope: per-user vs tenant-wide? Proposal assumes tenant-wide for staff roles, patient-scoped view comes free via entity refs.
- Real-time vs poll: start with paginated fetch; SSE/WebSocket later if needed.
