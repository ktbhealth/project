+++
id = "P-054"
title = "Role plugin: dynamic roles assignable to users and groups"
status = "done"
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
- `role_permissions` — `(role_id, resource, action, effect: "allow" | "deny")`. `resource`/`action` come from the app's permission vocabulary (see below); the package validates shape, not meaning.
- `subject_roles` — `(subject_kind: "user" | "group", subject_id, role_id, tenant_id?, granted_by, expires_at?)`. `tenant_id NULL` = global grant; set = grant applies only inside that tenant (P-053) — this is what makes "clinic-admin *of clinic X*" expressible without a parallel model. Group assignments resolve through `group_members`: effective permissions = user's direct roles ∪ roles of every group the user belongs to, filtered to the active tenant.

**Permission vocabulary** — generated, not hand-maintained: a build/codegen step derives the `(resource, action)` catalog from the app's permix `Def` plus an explicit `platformActions` list, emitting typed constants. Role editing UI and `Permissions.can` calls use the same catalog — no stringly-typed drift, autocomplete preserved.

**Resolution** — `Permissions` Effect service bound via `Principal`/`runAs` (extended to carry `Tenant` per P-053): `permissionsFor(userId, tenantId)` → effective grant set, `can(resource, action)` boolean, `requirePermission` failing `Forbidden`. Cached **per request** only.

**Cache invalidation** — a `permission_epoch` counter bumped on every write to `role_permissions`, `subject_roles`, or `group_members`; cached grant sets are keyed `(userId, tenantId, epoch)` so membership/role changes propagate on the next request without explicit purging. `expires_at` is evaluated at resolve time — expired grants simply stop resolving.

**Enforcement — explicit precedence, per-entity opt-in.** Two authz systems exist; the rule is:

- Each entity declares `authz: "static" | "rbac"` (default `static`).
- `static` → permix `rulesFor` only (today's behavior, unchanged).
- `rbac` → dynamic grants decide: effective = union of allow grants **minus** deny grants (deny always wins), still gated by the tenant role from P-053. Static permix rules for that entity are bypassed *only* in rbac mode — never silently widened, adoption is per-entity and explicit.
- Tenant scoping (P-053 `tenantScope`) applies underneath both modes — RBAC decides *capability*, tenant decides *visibility*; neither substitutes for the other.

**Plugin surface** — `plugins/rbac` exposes `RoleQueries`/`RoleCommands` via `@effect-server-utils/cqrs` (list/create/update roles, grant/revoke permissions incl. deny, assign/unassign subjects with optional tenant scope and expiry) and a generic `panel.pages` admin page (role list, permission matrix editor, subject assignments).

## Application to ktbhealth/product

- Vocabulary seeded from permix `Def` (`patient.read`, `consultation.update`, …) + platform actions (`billing.manage`, `members.invite`).
- Clinic staff roles (`doctor`, `nurse`, `receptionist`, `auditor`) become data — clinics customize without deploys.
- `subject_roles.tenant_id` scopes "clinic-admin" grants to the clinic's tenant; group assignments ride on P-032 groups.
- Migration path: entities flip `static → rbac` one at a time; `patients`/`consultations` stay static until clinic pilots demand it.

## Why this shape

- Storage + resolution generic; vocabulary generated from permix `Def` so the two authz layers share one source of truth.
- Deny-overrides + per-entity opt-in means RBAC can only ever *narrow or explicitly extend* what an entity declares — no accidental privilege escalation vs static rules.
- Epoch-keyed cache gives correct invalidation across group-membership churn without distributed cache infrastructure (works identically on sqlite/D1/pg — single-writer DBs).
- Group inheritance + tenant-scoped grants reuse P-032/P-053 instead of parallel models.

## Open decisions

- Permission model: flat `(resource, action)` strings vs hierarchical resources (`patient:123.read`) — start flat, scope via tenant/entity args at check time.
- Role templates/presets shipped by apps vs fully user-defined.
- `expires_at` in v1 or deferred (schema already carries it).
- Whether `Permissions` merges into `@ktb/authz` or stays its own package.
- Cross-tenant group roles: can a group span tenants (clinic group assigned a role valid in tenant A *and* B)? Likely yes via multiple `subject_roles` rows — confirm at implementation.
