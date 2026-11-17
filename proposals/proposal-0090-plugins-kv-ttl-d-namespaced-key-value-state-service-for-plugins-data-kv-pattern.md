+++
id = "P-090"
title = "plugins/kv: TTL'd namespaced key/value state service for plugins (data.kv pattern)"
status = "done"
created = "2026-11-17"
owner = "devin"
priority = "med"
links = ["P-053", "P-084", "P-089"]
done = "2026-11-17"
+++

## Problem

Plugins need small durable key/value state — per-tenant feature flags, rate-limit counters, idempotency keys, job checkpoints, wizard drafts — but today the only primitive is `plugin_state` (@ktb/db, P-058): one row per plugin, deployment-global, config-JSON only, no TTL, no namespacing, no per-tenant scoping (P-053 pending). Anything more means hand-rolled tables per plugin → schema churn for trivial state.

## Proposal

`plugins/kv` (`@ktb/kv`): a `data.kv` pattern plugin — sibling to `document-store` (P-089) in the persistence-pattern family.

- Effect service `Kv` with namespaced keys: `get/set/delete/increment/expire`, scoped by `{ plugin, tenant?, owner? }` — tenants get real fencing via the same tenant mechanism as documents.
- Values: JSON payloads validated per-key against an optional field DSL/schema registered by the owning plugin (untyped `json` allowed for opaque blobs).
- TTL first-class: `set(key, value, { ttl })`, lazy expiry on read + opportunistic sweep; `increment` for counters (rate limits, quotas).
- Storage: ONE drizzle table `kv_entries` (namespace/plugin_id, key, tenant_id, owner_id, value JSON, expires_at, version for CAS) — no per-plugin DDL, ever.
- Compare-and-swap via `version` for safe concurrent updates; `setIfAbsent` for idempotency keys / leader election-ish flags.
- Optional in-isolate L1 cache with stamp-based invalidation (mirror I18nRuntime's cache pattern) for hot reads.

## Non-goals

- Not a document store — no queries, no indexes beyond key lookup. Collections with fields/relations → P-089.
- Not a session/cache layer for auth — better-auth owns sessions.
- Not Cloudflare KV/Durable Objects — this is durable state in the existing D1/SQLite plane (ADR-0004 portability preserved; a CF-KV adapter could come later behind the same port).

## Relations

- Family: P-089 (document-store is pattern #1; kv is the degenerate/simple sibling — ships only if it earns its keep vs. documents-with-ttl).
- Absorbs the need behind P-053's per-tenant plugin_state scoping for arbitrary keys.
- Candidates: P-008/P-009 (appointment reminder dedupe keys), P-031 (doctor QoL state), rate limiting for public endpoints (patient portal invites).
