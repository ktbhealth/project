+++
id = "P-057"
title = "Platform console UX: shared page primitives + CRUD on all entity pages"
status = "done"
created = "2026-11-11"
owner = "devin"
priority = "high"
links = ["T-032", "T-034"]
epic = "platform"
+++

## Why

The /platform console pages each hand-roll their own header, tables and actions. Inconsistent UX, no shared density/pagination patterns, destructive actions without confirmation, and several entities are list-only despite domain services existing.

## What

Phase 1 — @ktbhealth/ui primitives (tag v0.2.0):
- `PageHeader` (hero): eyebrow, title, description, actions slot, optional back link.
- `DataToolbar`: search field, filters slot, result count, actions slot.
- `TableFooter` / pagination meta: "Showing x–y of z", page-size, prev/next.
- `ConfirmDialog`: AlertDialog wrapper for destructive mutations (async confirm, busy state, error slot).
- `PageContainer`: consistent max-width + vertical rhythm for console pages.

Phase 2 — product: bump @ktbhealth/ui to the new tag across plugins; apply PageHeader + DataToolbar + TableFooter to all six /platform pages (dashboard, consultations, groups, patients, roles, users).

Phase 3 — CRUD completion per entity:
- roles: edit dialog (updateRoleFn exists unused), AlertDialog on delete role/permission revoke/assignment removal.
- groups: create group, add/remove members via existing Groups service (needs fns), AlertDialog on member removal.
- users: suspend/reactivate if auth supports it; otherwise detail view + role-assignment shortcut.
- patients: admin edit via existing updatePatientFn or deep-link to record page.
- consultations: read-only list + detail link (no admin CRUD by design).

Phase 4 — verify: typecheck, 100% coverage gate, manual e2e on dev server, CI green, merge.

## Constraints

- Components live in repos/ui only (source of truth); release via git tag.
- 100% coverage gate applies to product; ui has its own test conventions.
- Dangerous actions always go through ConfirmDialog.
