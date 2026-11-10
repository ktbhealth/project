+++
id = "P-053"
title = "Tenant plugin: generic request-scoped tenancy for any kernel app"
status = "open"
created = "2026-11-10"
owner = "devin"
priority = "high"
links = ["P-031", "P-032", "P-050"]
epic = "platform"
+++

## Context

The kernel/plugin architecture (P-031–P-050) gives every app the same primitives: plugins, CQRS APIs, request-scoped `Principal`, permix authz. What it lacks is tenancy — nothing resolves *which* tenant a request acts in, and nothing scopes data to one. Today `ktbhealth/product` works around this with per-user ownership (`patients.ownerId`, `rulesFor` in `@ktb/authz` grants only the owner), which does not scale to clinics/orgs — and every future app on the kernel would reinvent it.

This proposal is deliberately **generic**: the capability lives in the platform layer and any app opts in. KTB Health is the first consumer, not the design target.

## Proposal

A `@ktb/tenant` package (`packages/tenant`) providing domain-agnostic tenancy primitives:

**Tenant context** — `Tenant` `Context.Reference`: `{ id, kind, slug, role } | null`, bound per request by the composition root next to `Principal`. `null` = personal scope; system contexts (webhooks, jobs) stay `null`. Apps may extend the payload via a generic `meta` slot — the package knows nothing about clinics or patients.

**Resolution port** — `TenantResolver` Effect service (port): given the request context, returns the active tenant. Ships with composable strategies the app wires at the edge:

- explicit selector (`?tenant=` param / header / subdomain — app picks transport),
- membership lookup (defaults to a `TenantDirectory` port the app implements — e.g. backed by the groups plugin's API, never imported directly),
- fallback policy (personal default vs deny).

**Scoped data** — a `tenantScope` combinator/helper: repos declare `tenant_id` (convention over configuration) and queries get the predicate injected; personal rows keep `tenant_id = null`. Works for any table shape — no per-domain code in the package.

**Authz factory** — `tenantAuthz(rules)` builds permix rules from a role→action matrix the *app* supplies; the package owns role semantics (owner/admin/member) and `Forbidden` behavior.

**Optional plugin surface** — a `plugins/tenant` contribution exposing `TenantQueries`/`TenantCommands` (resolve, listMine, switch) via `@effect-server-utils/cqrs`, plus a generic `panel.pages` admin page (members, roles, invites) any app can mount or hide.

## Application to ktbhealth/product

- `patients`, `consultations`, … gain nullable `tenant_id`; `rulesFor` swaps owner-only checks for the tenant-aware factory.
- Tenant kinds: `"clinic"`, `"org"`; membership/roles ride on P-032 `groups`/`group_members` through the `TenantDirectory` port.
- Selector transport + default-tenant policy decided at the composition root.

## Why this shape

- Port-adapter throughout: package defines ports (`TenantResolver`, `TenantDirectory`); apps and plugins provide adapters. No plugin→plugin imports.
- Mirrors `Principal`/`runAs` — one more request-scoped reference, zero changes to existing binding.
- Generic by construction: kinds, roles matrix, selectors, and tables are all app-supplied; the package only owns resolution, scoping, and authz mechanics.

## Open decisions

- Package (`packages/tenant`) vs plugin-only (`plugins/tenant`) split — likely both: package = ports/helpers, plugin = CQRS + admin page.
- Selector transport: header vs subdomain vs session-persisted choice.
- Whether `Tenant` folds into `Principal` or stays a separate reference.
- Backfill/migration strategy for apps with existing owner-scoped rows (product's `ownerId` data).
