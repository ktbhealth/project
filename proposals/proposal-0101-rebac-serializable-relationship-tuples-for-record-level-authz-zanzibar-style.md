+++
id = "P-101"
title = "ReBAC: serializable relationship tuples for record-level authz (Zanzibar-style)"
status = "open"
created = "2026-11-19"
owner = "devin"
priority = "high"
links = ["P-054", "P-084"]
+++

## Context

`packages/rbac` is flat RBAC (roles → `(resource, action)` grants) plus `packages/authz` static ownership rules (`c.userId === userId`). This covers today's gates but cannot express relationship questions the product will need: "covering physician sees this patient's record this week", "clinic staff share a consultation", "specialist consults on a case". LuckyPerms-style direct grants/wildcards are conveniences; the expressive gap is **relationships**.

Requirements: serializable (tuples as data in D1/SQLite), runtime-evaluated (per-request Effect service), relationship-native, expressive via userset rewrite — i.e. Zanzibar-style ReBAC.

## Proposal

Self-hosted ReBAC (OpenFGA/SpiceDB are gRPC services — do not fit Workers; Cedar is attribute-policy, not relationship-graph).

### Model

One table, one service:

```
relation_tuples(
  object_type, object_id, relation,   -- "patient" "42" "reader"
  subject_kind, subject_id,           -- "user" "alice" | "group" "clinic-staff" | "role" "doctor"
  tenant_id NULL,                     -- tenant-scoped tuples never leak (P-084 semantics)
  expires_at NULL,                    -- reuse subject_roles expiry machinery
  created_by, created_at
)
```

```
user:alice ──member──▶ group:clinic-staff ──staff──▶ clinic:main
patient:42  ──owner──▶ user:bob
consultation:7 ──covering──▶ group:clinic-staff (expires 2026-11-20)
```

### Check path

`Relations.can(object, relation, subject)` resolves via bounded userset expansion:

1. Direct tuples on `(object, relation)`.
2. Userset tuples: `object#relation@group:X` → expand `group:X#member` → users.
3. Role bridge: RBAC `can()` stays the coarse layer (`platform.admin`, `rbac.manage`); entity checks migrate to relations per-entity via the existing `EntityAuthzMode` seam — add `"relations"` as a third mode.
4. Tenant filter + `expires_at` honored; epoch-style cache keyed on a `relation_epoch` counter (same pattern as `permission_epoch`).

### Enforcement

`requireEntityAccess` gains `mode: "relations"` → `Relations.requireCan(...)`. Permix `rulesFor` ownership becomes the degenerate case `patient:*#owner@user:*` — migrated per-entity, default stays `static`.

## What this subsumes

- Wildcard permissions → userset subjects (`group:*` / typed subjects)
- Role inheritance → relation edges (`role:senior-doctor#includes@role:doctor`)
- Direct user grants → direct tuples
- Temporary/contextual perms → `expires_at` + tenant/portal scope columns

## Non-goals

- No SpiceDB/OpenFGA dependency — tuples live in our D1 schema, resolver is ~bounded BFS over one table.
- RBAC stays for platform-level gates; relations layer is additive.
- No snapshot/zookie machinery — checks run in the request scope; writes bump epoch (same consistency model as `subject_roles` today).

Refs: P-084 (tenant-scoped grant semantics), P-054 (permission system), `packages/rbac`, `packages/authz`, ADR-0003.
