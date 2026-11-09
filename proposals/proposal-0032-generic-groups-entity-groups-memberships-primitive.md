+++
id = "P-032"
title = "Generic groups entity: groups + memberships primitive"
status = "open"
created = "2026-11-09"
owner = "devin"
priority = "medium"
epic = "platform"
+++

## Context

Today every owned entity hangs off a bare `ownerId`/`userId` FK to the
better-auth `user` table (`patients.ownerId`, `consultations.userId`).
There is no way to express "these users share a patient base" (clinic),
"these users belong to an org" (organization), or "these patients form a
cohort". The platform console (`/platform`) is now plugin-driven and
ready to host a Groups admin surface.

We need the *generic* primitive first — clinics, organizations and
patient cohorts are all specializations of the same shape and should
be rows/metadata, not new tables.

## Proposal

New `plugins/groups` package (`@ktb/groups`) contributing:

**Tables** (via `tables` contribution → `packages/db/src/schema.ts`):

- `groups` — `id` (integer pk), `name`, `slug` (unique), `kind` (text,
  free-form: `"clinic"`, `"org"`, `"cohort"`, …), `meta` (JSON text,
  nullable — kind-specific payload), `created_at`, `updated_at`.
- `group_members` — `id`, `group_id` FK → `groups.id` cascade,
  `member_kind` (`"user"` now; `"patient"` later), `member_id` (text),
  `role` (nullable: `"owner"` | `"admin"` | `"member"`), `created_at`;
  unique on `(group_id, member_kind, member_id)`.

**Service** — `Groups` Effect service, swappable in tests:

- Repo helpers: `createGroup`, `listGroups`, `getGroup`, `addMember`,
  `removeMember`, `listMembers`, `findMembership(groupId, kind, id)`.
- Service methods: `create`, `list`, `get`, `addMember`, `removeMember`,
  `members`, `requireMember` (fails `Forbidden` unless the bound
  principal is a member — the hook future authz rules need).
- Admin ops gated by `PlatformAdmin.requireAdmin()` (same pattern as
  `Patients.adminList`); membership checks use `Principal`.

**Console** — `groups.panel` sidebar entry (`/platform/groups`, "Groups"
group or Access) + `AdminGroupsPage` view (dense table: name, slug,
kind, members count, created) + `adminListGroupsFn` server fn +
physical route module under `apps/web/src/routes/platform/`.

## Design notes

- **Polymorphic membership** (`member_kind` + `member_id`) keeps one
  join table for users today and patients tomorrow — a patient cohort
  is just `kind="cohort"` + `member_kind="patient"` rows. Trade-off:
  no FK enforcement on `member_id` and kind-specific queries need a
  `member_kind` filter; accepted for genericity.
- **`kind` + `meta` over typed tables**: clinics/orgs/cohorts differ in
  metadata (CNPJ? program? schedule?), not in structure. Specializations
  later register kind validators/metadata schemas in their own plugins.
- **No schema coupling to authz yet**: `groups` is not a permix
  resource in phase 1 — console access is the env allowlist; member
  checks are service-level `requireMember`.

## Non-goals (phase 1)

- No `group_id` on `patients`/`consultations`, no authz rule changes,
  no data scoping — ownership stays `ownerId`.
- No end-user UI (invite flows, member management pages) — console
  only.
- No billing/plan entitlements, no slug-based public URLs, no soft
  delete.

## Phase 2 candidates

- `group_id` columns + permix rules (`patient.read` scoped to group
  membership) — the clinic use-case.
- Typed kind registry: plugins declare `groupKind` metadata schemas.
- Cohort queries: `member_kind="patient"` listings feeding reports.

## Open questions

- `member_id` as text vs per-kind FK tables (recommend text +
  `member_kind`, revisit if referential integrity bites).
- Should `groups.slug` be unique per kind or globally?
- Does a user auto-join a "personal" group on signup (implicit tenant)
  or stay groupless until invited?
