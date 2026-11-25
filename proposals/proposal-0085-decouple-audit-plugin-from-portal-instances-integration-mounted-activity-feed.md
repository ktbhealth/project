+++
id = "P-085"
title = "Decouple audit plugin from portal instances — integration-mounted activity feed"
status = "done"
created = "2026-11-16"
priority = "med"
+++

## Problem

P-080 shipped `/portal/clinician/activity` by having `plugins/audit` declare a `pages` contribution hardcoded to the `clinician` portal instance (`path: "/portal/clinician/activity"`, chrome `portal: "clinician"`). The audit plugin now knows about a specific workspace portal — backwards coupling. P-081 (portal plugin: audience-scoped portal instances) will multiply instances (secretary, translator, …); each hardcoded mount rots, and audit cannot serve portals that don't exist yet.

## Proposal

Audit stays a pure domain plugin: table, `Audit` service (`record`/`adminList`/`adminVerify`/`activityAccess`/`activityList`), the reusable `AuditPage`/`AuditActivityPage` views, i18n catalogs. It stops declaring workspace `pages`/`chrome` contributions for `/portal/*` entirely.

Mounting becomes the consumer's job — the integration pattern already used for ports:

- `plugins/audit` exports the activity page component via a client-safe subpath (e.g. `@ktb/audit/activity-page`) so portal plugins can reference it from their `client.ts` mirrors.
- Each portal plugin that wants the feed contributes its own `pages` entry (`<basePath>/activity`, guard → `Audit.activityAccess()`, loader → `Audit.activityList()` catching `Forbidden` → empty) and its own `portal.sidebar` chrome item (group `Governance`). `plugins/clinician` is the first and only consumer today.
- The audit plugin keeps `/platform/audit` — the platform console is host infrastructure, not a tenant workspace; that mount is legitimate.

## Boundaries

- No new plugin-kernel mechanism (no "portal declares feeds" config) for v1 — plain per-portal page+chrome contributions. A generic mount earns its keep only when a second portal actually wants the feed.
- `Audit` service API unchanged; `activityAccess`/`activityList` remain the single read contract.
- Plugin dep edges: portal plugins get `optionalDeps: [auditServicesPlugin.id]` (ordering hint; absence = page guard denies/empty feed, plugin still works).

## Open questions

- Component export shape: new `@ktb/audit/activity-page` subpath vs re-export through `@ktb/kernel`-safe module.
- Whether the sidebar item label/group should be localized per portal (host `pluginPanelLabel`/`pluginPanelGroup` already translates by `to` path — likely sufficient).
