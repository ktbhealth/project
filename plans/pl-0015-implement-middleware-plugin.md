+++
id = "PL-015"
title = "Implement middleware plugin"
status = "open"
created = "2026-11-20"
priority = "med"
links = ["P-106", "P-110", "PL-011"]
+++

# Plan: middleware plugin (P-110)

## Phase 1 — Pipeline core
- Define middleware contribution contract in kernel: `{ id, order, scope(routeKey|context|path glob), handler }`.
- Host edge runner in `@ktb/app-host`: resolve effective chain per request, execute in order, early-return on rejection.
- Keep services pure — middleware is adapter-edge composition, arch-lint exempt like other edge code.

## Phase 2 — Built-in stages
- Principal/session resolution, tenant context, locale.
- Platform-admin gate as a stage (composes with `PlatformAdmin` service — service keeps `requireAdmin` for defense-in-depth inside domain services).

## Phase 3 — Migrate existing guards
- Descriptor `guard` callbacks → generated from middleware where mechanical (audit first, don't break contracts).
- `portalAdmits` audience checks evaluated: absorb into middleware or keep portal-level — decide with evidence.

## Phase 4 — Admin page
- `/platform/middleware`: stages table (id, order, scope, plugin, enabled) + per-route effective chain inspector.
- Read-only v1; toggles later if safe.

## Phase 5 — Plugin-contributed stages
- rbac grant check, captcha, rate-limit as plugin contributions proving extensibility.

## Phase 6 — Verify
- Order determinism tests; rejection paths (401/403/redirect) per stage; admin page reflects real registry.
