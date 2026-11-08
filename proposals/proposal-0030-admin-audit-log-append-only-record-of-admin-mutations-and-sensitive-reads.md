+++
id = "P-030"
title = "Admin audit log: append-only record of admin mutations and sensitive reads"
status = "done"
created = "2026-11-08"
priority = "medium"
labels = ["admin"]
epic = "admin"
+++

## Idea

Admin audit log: an append-only record of every admin mutation —
who (admin identity), what (action + target entity), when, from where.
Rendered at `/admin/audit` with filters by actor, action, target,
date. Also the access log for sensitive reads (patient detail views).

## Likely scope

- `admin_audit` table (actor, action, target_type/id, payload
  summary, ip/ua, created_at)
- Write helper called by every admin mutation + sensitive read
- `/admin/audit` list view with filters

## Why it matters

Admin panels concentrate power; without an audit trail a rogue or
compromised admin account is invisible. LGPD also expects accountability
for access to patient data.

## Positioning cautions

- Append-only — no updates/deletes through the app.
- Log the action and target, not full clinical payloads.

## Depends on

- P-025 shell (lands with first mutation), authz actor identity
