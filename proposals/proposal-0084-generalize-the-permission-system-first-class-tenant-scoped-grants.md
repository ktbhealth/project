+++
id = "P-084"
title = "Generalize the permission system: first-class tenant-scoped grants"
status = "open"
created = "2026-11-16"
priority = "high"
+++

## Context

Surfaced by the i18n platform (ADR-0003, product PR #77). `packages/rbac` grant resolution is `tenant_id IS NULL OR tenant_id = <ctx>` — a GLOBAL grant silently satisfies every tenant-scoped check. For i18n this was confirmed as intended (global i18n editors may write any tenant's copy), but it exposes a general gap: **there is no way to grant a permission for exactly one tenant** without either a platform-admin role or a global grant that bleeds everywhere.

`plugins/i18n/src/admin.ts` `requireScopePermission` is the first consumer that needed scope-aware authz; it works around the limitation (tenant scope → `requirePermission(..., {tenantId: scopeId})`, portal → platform admin only) but the semantics live in a docstring, not the model.

## What needs deciding

1. **Audit** every `requirePermission`/`can` call site for implicit global-grant-bleed into tenant contexts — which plugins actually have tenant-scoped resources today vs which just inherit global semantics by accident.
2. **Semantics**: does global-inheritance stay the default (with an opt-in strict mode), or should tenant-scoped checks require a non-NULL-tenant grant? Per-resource policy, probably.
3. **Model**: make scope (global / tenant / portal) explicit in `PermissionOptions` rather than inferred from a bare `tenantId`; consider a scope-chain resolver mirroring i18n's portal→tenant→global.
4. **Apply**: i18n `requireScopePermission` becomes the reference implementation; other plugins with tenant data (patients, consultations, tenant plugin itself) get reviewed against the new semantics.

## Non-goals

- No behavior change for global grants (confirmed: global holders write any tenant copy).
- Not an RBAC rewrite — evolve `packages/rbac` incrementally behind the existing port.

Refs: ADR-0003, PL-008, product PR #77, `packages/rbac/src/index.ts`, `plugins/i18n/src/admin.ts`.
