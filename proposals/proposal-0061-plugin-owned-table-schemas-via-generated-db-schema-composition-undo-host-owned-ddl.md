+++
id = "P-061"
title = "Plugin-owned table schemas via generated db-schema composition (undo host-owned DDL)"
status = "in-progress"
created = "2026-11-11"
priority = "med"
labels = ["plugins"]
links = ["T-085", "P-058"]
epic = "platform"
+++

## Context

P-058 follow-up moved every plugin table host-owned into `packages/db/src/` to kill the `@ktb/db` ↔ plugin cycle (db re-exported plugin schemas while plugins imported `Db`). Works, zero drift — but table DDL is no longer colocated with the plugin that owns the domain. User wants plugin-owned schemas back, done prod-grade.

## Root cause

`@ktb/db` does two jobs: (1) provides the `Db` service/`DbClient` port, (2) composes all plugin schemas into one drizzle entry. Job (2) forces `db → plugin` package deps while plugins need `plugin → db` for `Db`. Circular.

## Proposal

Split the jobs:

- **Table definitions return to plugins**: `plugins/<name>/src/schema.ts` (and `packages/{auth,rbac}/src/schema.ts`) own their `sqliteTable` defs again; plugin code imports `./schema.ts` like before P-058-followup.
- **`@ktb/db` slims to the port**: `Db` service, `DbClient`, `makeDb`, testing helpers — zero org-internal deps (current state already).
- **Schema composition becomes a generated artifact outside @ktb/db**: `tools/gen-db-schema.mjs` scans `plugins/*/src/schema.ts` + `packages/*/src/schema.ts` (the existing discovery convention) and emits e.g. `packages/db-schema/src/schema.gen.ts` (new tiny package `@ktb/db-schema`, or `apps/web`-adjacent location — pick whichever keeps drizzle-kit + tests happy). `packages/db/drizzle.config.ts` points its schema entry at the generated file. Generated files are never hand-edited; add a `gen:db-schema` script + CI drift check mirroring `ci:route-types`.
- Dep graph becomes acyclic: `plugin → @ktb/db` (service) and `@ktb/db-schema → plugins + @ktb/db` (composition only). Nothing imports `@ktb/db-schema` at runtime except drizzle-kit/migration tooling and tests needing the composed set.

## Details to nail in implementation

- Which modules count as "schema providers": `plugins/*/src/schema.ts`, `packages/{auth,rbac}/src/schema.ts` — anything else? (interop? tenant?) Audit current `packages/db/src/*.ts` host files: patients/consultations/groups/notifications/auth/rbac/plugin-state. `plugin_state` is host-owned platform state — decide: keep in @ktb/db (it's not a plugin domain) or move composition-wise (it's fine host-owned; the generator can also emit/merge a hand-written `packages/db/src/plugin-state.ts` — or keep plugin-state.ts inside the generated package's own file).
- Non-table exports currently in plugin schema.ts files (types, `$inferSelect`, relations) move back with the tables; `@ktb/rbac` re-export shim pattern may need restoring.
- Tests importing tables: plugin tests go back to `./schema.ts`; apps/web/db tests import the composed entry.
- drizzle-kit generate must produce zero diff (identical schema).
- Naming: `@ktb/db-schema` package vs `apps/web` internal — prefer a package so plugins' own vitest configs and db:generate share one entry.

## Non-goals

- Runtime plugin table registration (impossible — drizzle migrations are build-time).
- Changing the `tables` Plugin contribution semantics (still declarative metadata).
