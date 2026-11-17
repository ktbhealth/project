+++
id = "P-089"
title = "plugins/document-store: runtime JSON-schema entity store for plugin-contributed data"
status = "done"
created = "2026-11-17"
owner = "devin"
priority = "med"
links = ["P-064", "P-065", "ADR-0004"]
done = "2026-11-17"
+++

## Problem

Plugin data definitions are compile-time drizzle tables aggregated into `packages/db/src/schema.gen.ts` and migrated by drizzle-kit at build/deploy. This blocks runtime-contributed data: specialty/locale packs (P-065), clinic custom fields, and third-party plugin data all require a schema change + migration + redeploy.

## Proposal

`plugins/document-store` (`@ktb/document-store`): a generic, runtime-validated document store plugin.

- Plugins declare collections as **data** via a typed field DSL (`f.string()`, `f.number()`, `f.boolean()`, `f.date()`, `f.enum()`, `f.json()`, `f.ref()`) carried on the generic `contributions` channel as kind `document.collection` — no kernel changes needed.
- Storage: two drizzle tables owned by the plugin — `document_collections` (registry: owner plugin, schema JSON, version) and `document_records` (collection, id, JSON payload, owner/tenant columns). No per-entity DDL → no migration engine; schema evolution = version bump + optional backfill.
- Service: Effect service `DocumentStore` — `ensure`/`insert`/`get`/`find`/`update`/`delete`. `find` supports mingo-lite filtering (`$eq $ne $gt $gte $lt $lte $in $exists $regex`, `$and`/`$or`), sort, limit/offset.
- `ref` fields are service-level references `{ collection, id }` validated on write, with `onDelete: restrict|cascade|nullify` enforced by the store — deliberately NOT DB-level FKs.
- Tenant/owner scoping columns mirror the patients pattern (`tenantId` nullable → personal row).

## Non-goals / mitigations

- Core relational domain tables (patients, consultations, authz, audit) stay on drizzle — this is for extensible/plugin-contributed data.
- Typed DSL gives compile-time field safety at call sites; storage stays schemaless JSON.
- The query engine is one bounded module — covered to the repo's 100% gate like everything else.

## Relations

- Enables: P-065 (specialty/locale packs contribute data), P-064 (runtime data plane alongside capability manifest), P-084/P-006 (consent-scoped dynamic data).
- ADR-0004: JSON documents are maximally DB-portable.
