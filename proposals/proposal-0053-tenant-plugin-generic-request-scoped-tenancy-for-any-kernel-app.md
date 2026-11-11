+++
id = "P-053"
title = "Tenant plugin: generic request-scoped tenancy for any kernel app"
status = "open"
created = "2026-11-10"
owner = "devin"
priority = "high"
links = ["P-031", "P-032", "P-050", "P-006"]
epic = "platform"
+++

## Context

The kernel/plugin architecture (P-031–P-050) gives every app the same primitives: plugins, CQRS APIs, request-scoped `Principal`, permix authz. What it lacks is tenancy — nothing resolves *which* tenant a request acts in, and nothing scopes data to one. Today `ktbhealth/product` works around this with per-user ownership (`patients.ownerId`, `rulesFor` in `@ktb/authz` grants only the owner), which does not scale to clinics/orgs — and every future app on the kernel would reinvent it.

This proposal is deliberately **generic**: the capability lives in the platform layer and any app opts in. KTB Health is the first consumer, not the design target.

## Proposal

A `@ktb/tenant` package (`packages/tenant`) providing domain-agnostic tenancy primitives:

**Tenant context** — `Tenant` `Context.Reference`: `{ id, kind, slug, role, meta } | null`, bound per request by the composition root next to `Principal`. `null` = personal scope; system contexts stay `null` until explicitly re-bound (see propagation below).

**Resolution plumbing** — `runAs(user, effect)` in `@ktb/app-host` today binds `Principal` + `AuthzLive` but never sees the request, so the tenant selector (header/subdomain/session) has no path in. Fix: the route layer extracts the selector and passes it into `runAs` (or a `TenantResolver` service in `BindingLive` resolves it during binding). Tenant resolution happens *inside* the per-request effect scope — never cached across requests.

**Resolution port** — `TenantResolver` Effect service: selector transport → membership lookup → fallback policy, composable. Membership defaults to a `TenantDirectory` port the app implements (e.g. backed by the groups plugin's API via port, never imported). Fallback: personal default vs deny — app chooses.

**Scoped data — three states, not two.** `tenant_id` semantics: `NULL` = personal, value = tenant-owned. Consent-shared rows (P-006: a patient history visible to *other* clinics) are **never** modeled by widening `tenant_id` — sharing is an explicit grant row in a separate link table, checked as an additional permission, so tenant scope can never leak through the scoping column itself.

**Enforcement — portable baseline + progressive enhancement.** Tenant isolation must hold on SQLite, D1, *and* Postgres:

- **Baseline (all engines):** a scoped-repo factory — `scopedRepo(table)` returns a query builder pre-filtered to the caller's accessible tenant set (`tenant_id IN (…)` OR `tenant_id IS NULL` for personal rows). Tenant tables may only be queried through it; an arch-lint rule (`ktb-arch/*`) bans raw queries on declared tenant tables. This is the non-bypassable layer — convention alone is not enough, since D1/SQLite have no RLS.
- **Enhancement (Postgres only):** auto-generated RLS policies per tenant table (`tenant_id = ANY(current_setting('app.tenant_ids')::int[])`), applied as defense-in-depth when the dialect is postgres; dialect-detected, no-op on sqlite/D1.
- **Test harness:** a per-plugin isolation suite that runs identical cross-tenant leak tests on every supported engine — catches guard bypasses in CI regardless of dialect.

**System-context propagation** — `run()` binds nothing, and the heaviest tenant flows are system flows (transcription pipeline P-015/P-016, webhooks, jobs). Rule: `tenant_id` travels explicitly on queue messages, webhook envelopes, and CQRS event metadata; system effects re-bind `Tenant` from the message payload via a `runAsTenant(tenant, effect)` helper. Ambient tenant is never inherited into background work — an unscoped message is a bug, not a default.

**Authz** — `tenantAuthz(rules)` builds permix rules from an app-supplied role→action matrix; package owns role semantics (owner/admin/member) and `Forbidden` behavior. Composes with P-054: tenant role gates first, dynamic RBAC grants second.

**Optional plugin surface** — `plugins/tenant` exposes `TenantQueries`/`TenantCommands` (resolve, listMine, switch) via `@effect-server-utils/cqrs` plus a generic `panel.pages` admin page (members, roles, invites).

## Application to ktbhealth/product

- `patients`, `consultations`, … gain nullable `tenant_id` + index; `rulesFor` swaps owner-only checks for the tenant-aware factory.
- Tenant kinds: `"clinic"`, `"org"`; membership/roles ride on P-032 `groups`/`group_members` through the `TenantDirectory` port.
- Selector transport + default-tenant policy decided at the composition root.
- Async transcription/audio jobs carry `tenant_id` on the queue message from day one.

## Why this shape

- Port-adapter throughout: package defines ports (`TenantResolver`, `TenantDirectory`); apps and plugins provide adapters. No plugin→plugin imports.
- Mirrors `Principal`/`runAs` — one more request-scoped reference; resolution plumbing is the only host change.
- Generic by construction: kinds, roles matrix, selectors, tables all app-supplied; the package owns resolution, scoping enforcement, and authz mechanics.
- Multi-engine by construction: portable guard is the contract; RLS is a bonus where available, never a requirement.

## Open decisions

- Package (`packages/tenant`) vs plugin-only (`plugins/tenant`) split — likely both: package = ports/helpers, plugin = CQRS + admin page.
- Selector transport: header vs subdomain vs session-persisted choice.
- Whether `Tenant` folds into `Principal` or stays a separate reference.
- Backfill/migration for existing `ownerId` rows (product): dual-read period vs one-shot migration; personal rows stay `tenant_id = NULL`.
- `runAs` signature: explicit tenant param vs resolver-in-`BindingLive` — pick at implementation.
