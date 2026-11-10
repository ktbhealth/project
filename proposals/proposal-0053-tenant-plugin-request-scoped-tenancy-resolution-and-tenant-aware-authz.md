+++
id = "P-053"
title = "Tenant plugin: request-scoped tenancy resolution and tenant-aware authz"
status = "open"
created = "2026-11-10"
owner = "devin"
priority = "high"
links = ["P-031", "P-032", "P-050"]
epic = "platform"
+++

## Context

Every owned entity hangs off a single user: `patients.ownerId`, `consultations.userId`, and `rulesFor` in `@ktb/authz` grants only the owner. P-032 adds the generic groups/memberships primitive (`kind: "clinic" | "org" | "cohort"`) but nothing resolves *which* tenant a request acts in, and no domain plugin scopes rows to one. Clinics are the first real multi-user tenant — without a tenant primitive every domain plugin will bolt on its own scoping, inconsistently.

## Proposal

New `plugins/tenant` package (`@ktb/tenant`) contributing:

**Concept** — a tenant is a `groups` row with a tenant-ish `kind` (`"clinic"`, `"org"`); membership and roles come from `group_members` (`role`: owner/admin/member). No new membership tables.

**Tenant context** — a `Context.Reference` alongside `Principal` in `@ktb/authz` (or `@ktb/tenant`): `{ id, slug, kind, role } | null`, bound per request by `runAs` in `@ktb/app-host`. `null` = personal scope; system contexts (webhooks, jobs, bot channels) stay `null`.

**Resolution** — explicit selector first (`?tenant=` param / header / subdomain — pick one), then the user's memberships, then personal default. `TenantQueries.resolve` / `listMine` via `@effect-server-utils/cqrs`, same pattern as the groups plugin.

**Scoped data** — add nullable `tenant_id` to owned tables (patients, consultations, …); repos gain a tenant-scope filter layered on `ownerId`. Personal rows stay `tenant_id = null`. Backfill/migration strategy is an open decision.

**Authz** — extend the permix `Def`/rules: tenant members read/update tenant-scoped rows per role; owner-only actions (billing, member management, tenant settings) gate on `role = "owner"`. `Forbidden` semantics unchanged.

**Platform surface** — tenant admin page via `panel.pages` (P-050): members, roles, invites.

## Why this shape

- Reuses the P-032 primitive — tenants are rows + metadata, not a parallel hierarchy.
- Request-scoped context mirrors `Principal`; zero changes to how plugins bind authz today.
- Domain plugins consume the `Tenant` reference and `tenantScope` helpers — no cross-plugin imports (port rule holds).

## Open decisions

- Tenant = `groups` row vs dedicated `tenants` table (slugs, billing binding).
- Selection mechanism: header vs subdomain vs session-persisted choice.
- Backfill: dual-read period vs one-shot migration of existing `ownerId` rows.
- Extend `Principal` with `tenantId` vs separate `Tenant` reference.
