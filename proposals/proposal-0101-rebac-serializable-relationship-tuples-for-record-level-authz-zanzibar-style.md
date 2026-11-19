+++
id = "P-101"
title = "ReBAC: serializable relationship tuples for record-level authz (Zanzibar-style)"
status = "in-progress"
created = "2026-11-18"
priority = "high"
links = ["P-054", "P-084"]
+++

## Context

`packages/rbac` is flat RBAC (roles → `(resource, action)` grants) plus `packages/authz` static ownership rules (`c.userId === userId`). This covers today's gates but cannot express relationship questions the product will need: "covering physician sees this patient's record this week", "clinic staff share a consultation", "specialist consults on a case". LuckyPerms-style direct grants/wildcards are conveniences; the expressive gap is **relationships**.

Requirements: serializable (tuples as data in D1/SQLite), runtime-evaluated (per-request Effect service), relationship-native, expressive via userset rewrite — i.e. Zanzibar-style ReBAC.

**Key decision: single model.** RBAC is expressible as tuples — a role grant is just an edge. Rather than maintaining two authz semantics (`static`/`rbac` modes in `EntityAuthzMode`), RBAC is retired into the tuple model; `roles`/`role_permissions`/`subject_roles` migrate to `relation_tuples`. The mode seam collapses to one path.

## Proposal

Self-hosted ReBAC (OpenFGA/SpiceDB are gRPC services — do not fit Workers; Cedar is attribute-policy, not relationship-graph).

### Model

One table replaces `roles` + `role_permissions` + `subject_roles`:

```
relation_tuples(
  object_type, object_id, relation,   -- "patient" "42" "reader"
  subject_kind, subject_id,           -- user | group | role | platform-node
  effect,                             -- allow | deny  (deny always wins)
  tenant_id NULL,                     -- tenant tuples never leak (P-084 semantics)
  expires_at NULL,                    -- temporal grants ("covering until Friday")
  created_by, created_at
)
```

Everything is an edge:

```
user:alice        ──member────▶ role:doctor
role:doctor       ──perm──────▶ patient:*
role:admin        ──perm──────▶ platform:admin
user:alice        ──covering──▶ patient:42            (expires_at set)
group:clinic      ──staff──────▶ clinic:main
```

### Check path

`Relations.can(object, relation, subject)` — one indexed tuple query + bounded userset expansion (BFS, depth ≤ 6, memoized per request):

1. Direct tuples on `(object, relation)`.
2. Userset tuples: `object#relation@group:X` → expand `group:X#member` → users.
3. Role bridge is just more tuples: `role:doctor#perm@patient:*` reached via `user#member@role:doctor`.
4. Platform gates included: `check("platform","admin","use")` — `platform.admin` is a tuple like any other.
5. Tenant filter + `expires_at` honored; `relation_epoch` counter invalidates caches (same pattern as `permission_epoch`).

### Enforcement

```
service → requireCan("patient","42","read") → Relations.can() → Forbidden
```

`Principal`, `Forbidden`, tenant scoping, `expires_at`, audit (`created_by`) carry over. `requireEntityAccess` loses its mode parameter — always `Relations`. Permix `rulesFor` ownership becomes degenerate tuples (`patient:42#owner@user:alice`) written at entity creation.

### What this subsumes

- Wildcard permissions → `patient:*` object_id or typed subjects
- Role inheritance → `role:senior#includes@role:doctor` edges
- Direct user grants/negations → direct tuples with effect
- Temporary/contextual perms → `expires_at` + tenant/portal scope
- RBAC roles/groups/assignments → same table, migrated

## Migration

1. Add `relation_tuples` + `relation_epoch`; backfill from `roles`/`role_permissions`/`subject_roles` (`role:X#perm@<resource>:<action>`, `subject#member@role:X`).
2. `Relations` service behind existing `Permissions` port signature — call sites unchanged initially.
3. Migrate `requireEntityAccess` call sites; write ownership tuples at entity creation; retire permix `Def`/`rulesFor`.
4. Drop `roles`/`role_permissions`/`subject_roles`; `PermissionsLive`/`buildPermissions` replaced by `RelationsLive`.

## Non-goals

- No SpiceDB/OpenFGA dependency — tuples live in our D1 schema; resolver is bounded BFS over one table (~2 queries + in-memory walk per check).
- No snapshot/zookie machinery — checks run in request scope; writes bump epoch (same consistency model as `subject_roles` today).
- RBAC grant semantics preserved during migration (deny-wins, tenant isolation, expiry) — behavior parity first, expressiveness second.

Refs: P-084 (tenant-scoped grant semantics), P-054 (permission system), `packages/rbac`, `packages/authz`, ADR-0003.
