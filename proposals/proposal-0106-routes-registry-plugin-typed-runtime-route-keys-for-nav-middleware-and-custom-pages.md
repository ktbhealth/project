+++
id = "P-106"
title = "Routes registry plugin: typed runtime route keys for nav, middleware, and custom pages"
status = "open"
created = "2026-11-20"
priority = "med"
links = ["P-104"]
+++

## Vision

A `routes` (or `url`) platform plugin that makes route identity first-class: every addressable path gets a typed key, and nav links, middleware/guards, and custom pages reference the key instead of raw strings — type-safe URLs at runtime.

## Why

- Custom pages (`plugins/pages`) already claim paths via descriptors + a `/*` wildcard guard; route identity is implicit string matching.
- Nav (`app.header.nav`, `portal.sidebar`), middleware/guards, and cross-plugin links all hardcode `/platform/...` or app paths — renames break silently.
- Puck builder pages and templates will generate links too; they need a stable route contract.

## Shape (proposal)

- Registry of route definitions contributed by plugins: `{ key, path, params schema, portal/context }`.
- Helper `route("patients.detail", { id })` → typed path; compile-time keys via declaration merging (same pattern as `PluginRegistry`).
- Custom pages register their slug under a route key; wildcard guard resolves via registry instead of string path.
- Optional: route table admin page under `/platform` for inspection.

## Open questions

- Static codegen (TanStack Router already generates routeTree) vs runtime registry — likely complementary: registry for *dynamic* (DB) routes + plugin-contributed nav targets.
- Does this subsume or sit beside TanStack file routes? File routes stay the mechanism; registry adds identity/typing for contributed + DB routes.
