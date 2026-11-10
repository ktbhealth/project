+++
id = "P-054"
title = "Role plugin: dynamic roles assignable to users and groups"
status = "open"
created = "2026-11-10"
owner = "devin"
priority = "high"
links = ["P-032", "P-050", "P-053"]
epic = "platform"
+++

## Context

Authz today is static: `rulesFor(userId)` in `@ktb/authz` hardcodes owner-only rules per entity, and P-053's tenant factory covers role *within* a tenant (owner/admin/member) but not named, app-defined roles. There is no way to say "Dr. Silva is an `auditor`" or "the billing team can `invoice.read`" without code changes. Every app on the kernel needs runtime-managed RBAC: create roles, grant permissions, assign them to users and to groups (P-032).

This is deliberately **generic**: permission vocabulary and role lifecycle are app-supplied; the package owns storage, resolution, and enforcement mechanics.

## Proposal

A `@ktb/rbac` package (`packages/rbac`) plus optional `plugins/rbac` contribution:

**Model** — three primitives, all app-agnostic:

- `roles` — `id`, `name`, `description`, `meta` (app payload). Roles are data, created at runtime.
- `role_permissions` — `(role_id, resource, action)` grants. `resource`/`action` are strings from the app's permission vocabulary; the package validates shape, not meaning.
- `subject_roles` — `(subject_kind: "user" | "group", subject_id, role_id, granted_by, expires_at?)`. Group assignments resolve through `group_members` — effective permissions = user's direct roles ∪ roles of every group the user belongs to (tenant-scoped when P-053 is present: `subject_kind` may gain `"tenant"` later).

**Resolution** — `Permissions` Effect service: `permissionsFor(userId)` → effective grant set (cached per request), `can(userId, resource, action)` boolean, `requirePermission` failing `Forbidden`. Plugs into the existing `Principal`/`runAs` binding — no signature changes for callers.

**Enforcement** — two layers, app chooses per check:

- static permix rules stay for compile-time entity authz (`rulesFor`);
- dynamic grants checked via `Permissions.can` inside services — the escape hatch for runtime roles. Composes with P-053: tenant role gates first, RBAC grants second.

**Plugin surface** — `plugins/rbac` exposes `RoleQueries`/`RoleCommands` via `@effect-server-utils/cqrs` (list/create/update roles, grant/revoke permissions, assign/unassign subjects) and a generic `panel.pages` admin page (role list, permission matrix editor, subject assignments).

## Application to ktbhealth/product

- Permission vocabulary seeded from existing permix `Def` (`patient.read`, `consultation.update`, …) plus platform actions (`billing.manage`, `members.invite`).
- Clinic staff roles (`doctor`, `nurse`, `receptionist`, `auditor`) become data — clinics customize without deploys.
- Group assignments ride on P-032 groups: give `clinic-admin` role to the clinic group → every member inherits.

## Why this shape

- Storage + resolution generic; vocabulary and lifecycle app-owned — same port-adapter split as `@ktb/tenant`.
- Group inheritance reuses P-032 instead of a parallel membership model.
- Dynamic layer sits *beside* static permix, not inside it — no rewrite of `rulesFor`; apps adopt incrementally.

## Open decisions

- Permission model: flat `(resource, action)` strings vs hierarchical resources (`patient:123.read`) — start flat, scope via tenant/entity args at check time.
- Deny-overrides vs union-only semantics; explicit deny grants.
- Role templates/presets shipped by apps vs fully user-defined.
- Expiring grants (`expires_at`) in v1 or deferred.
- Whether `Permissions` merges into `@ktb/authz` or stays its own package.
