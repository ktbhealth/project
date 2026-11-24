+++
id = "PL-011"
title = "Implement routes registry plugin"
status = "done"
created = "2026-11-20"
owner = "devin"
priority = "med"
links = ["P-106"]
epic = "site-builder"
done = "2026-11-24"
+++

# Plan: routes registry plugin (P-106)

## Phase 1 — Registry core
- Define route contribution kind in `@ktb/kernel` (or new `packages/routes`): `{ key, path, params?, portal/context }`.
- `Routes` service: register at boot, `route(key, params)` → typed path, `match(path)` → route key (for wildcard/custom-page resolution).
- Compile-time key typing via `PluginRegistry`-style declaration merging.

## Phase 2 — Plugin contributions
- Migrate existing nav/sidebar targets to registry keys (portal.sidebar `to:` fields, `app.header.nav` links).
- Custom pages (`plugins/pages`) register slug → route key; wildcard guard resolves via `match()` instead of raw path string.

## Phase 3 — Admin surface (optional)
- `/platform/routes` read-only table: key, path, contributing plugin, params schema.

## Phase 4 — Verify
- Type tests: unknown key fails compile; param mismatch fails.
- Runtime: nav links, middleware guards, custom page render all resolve via registry.
