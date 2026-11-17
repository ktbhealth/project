+++
id = "ADR-0004"
title = "Serverless now, DB portable by construction"
status = "open"
created = "2026-11-17"
owner = "devin"
priority = "high"
+++

# ADR-0004: Serverless now, DB portable by construction

## Decision

Stay on Cloudflare Workers + D1 + R2 + Queues + Durable Objects.
Do NOT migrate to containers or Postgres now. Instead, keep the
deployment topology swappable by construction: the binding layer is
already ~90% ported (verified by audit, Nov 2026); the remaining
lock-in is the drizzle-sqlite schema/migration stack, which we
defuse incrementally so a future Postgres swap is a driver change,
not a rewrite.

## Rationale

- Every edge binding already sits behind a port (`SqlDatabase`,
  `AudioStore`, `JobsEnqueuer`, `AiRunner`, `Email`, `Captcha`,
  `Bot`, `AnamnesisEngine`, `Telemetry`); plugins never touch
  `env`/bindings directly — verified by grep audit.
- `dev:node` already boots the app outside workerd with FS/SQLite
  stand-ins; `email-smtp` proves second-transport adapters work in
  both runtimes.
- The real lock-in is schema-level: 31 `plugins/*/src/schema.ts`
  files use `drizzle-orm/sqlite-core`, `SQLiteTable` is baked into
  `kernel/contributions.ts` and `tenant/scoped-repo.ts`, defaults
  use `unixepoch('subsecond')` (~20 files), `db.batch` transactions
  at 6 sites, better-auth pinned `provider:"sqlite"`, and
  `drizzle-kit` generates sqlite migrations only.
- `SqlDatabase` is SQLite-shaped (`?` params, `batch`, `exec`,
  `dump`) — Postgres does not fit that wire shape; the port's honest
  future is exposing the drizzle handle, not forcing PG into
  `prepare/bind`.
- `tenant/rls.ts` is already dialect-aware and PG-ready; the PG
  seam was designed in, just unused.
- `infra/ports` + `infra/oci` (Always-Free ARM VMs, Pulumi) already
  provide the multi-target infra seam and a host for Postgres if
  the day comes.
- The one true edge lock-in is `AcpBridge` (hibernating per-endpoint
  WebSocket Durable Object) — acceptable: it is optionalDeps-style
  and route-gated; a container path would use `ai-adapter-node`
  stdio transport instead.

## Consequences

Guards adopted now (cheap, prevent ossification):

1. `packages/db/src/table.ts` facade — single import point for the
   table DSL so a future `pg-core` port rewrites one file, not 35.
2. `unixepoch` defaults replaced by app-side timestamps.
3. `tx()` helper wraps the 6 `db.batch` sites; PG maps to
   `.transaction` without call-site changes.
4. `Db` service exposes the drizzle handle honestly; the one raw
   `prepare` in `ai-bridge`'s DO moves behind a port method.

When Postgres becomes real: new `packages/db-postgres` adapter +
`pg-core` schema generation + Hyperdrive binding — app/plugin layer
unchanged. When containers become real: port edge bindings
individually (Queue hardest, DO hardest), never big-bang.

Revisit triggers: Workers CPU/time limits under pilot load,
long-lived connection needs, or ACP bridge becoming core.
