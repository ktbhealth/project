+++
id = "P-059"
title = "Generic pages contribution: plugin-declared pages at arbitrary paths (supersedes panel.pages scope)"
status = "open"
created = "2026-11-11"
priority = "med"
labels = ["plugins"]
links = ["P-058", "PL-005"]
epic = "platform"
+++

## Context

Pages today come in three hardcoded shapes: `routes` plugins (physical() file-route subtrees — full TanStack features but build-time only), `landing` (single whole-page slot at "/"), and `panel.pages` (P-050 — path descriptors dispatched under /platform only, console context). There is no way for a plugin to contribute a page at an arbitrary app path, and nothing DB/runtime-driven can ever be a page — which blocks the custom-pages plugin (user-authored pages stored in DB) and forces every simple page through file-route ceremony.

## Proposal

A generic **`pages` contribution kind** on `Plugin`, generalizing `panel.pages` beyond /platform:

```ts
pages: [{
  id: string,
  path: string,                  // any absolute path
  component: ComponentType<{ data: unknown; ctx: PageContext }>,
  loader?: () => Effect<unknown, unknown, unknown>,  // runAs-bound
  guard?: () => Effect<boolean, unknown, unknown>,   // false → 404
  context?: "app" | "console",   // which chrome/layout wraps it (default "app")
}]
```

`PageContext` mirrors `PanelPageContext`: `{ plugins, locale, user? }` injected by the host — plugins never import apps or i18n runtime.

## Host mechanics

- TanStack routes are build-time, so the host adds splat dispatcher route(s) that resolve path → descriptor → guard → loader (`runAs`, Principal+Tenant+Authz bound) → component. `platform/$` is the existing proof; generalize to an app-level splat (e.g. `/$` or `/_/$`) — static file routes keep match priority, splat catches the rest.
- Per-page auth semantics declared via `guard`; hard authz stays inside loader/services as today.
- `panel.pages` folds into `pages` with `context: "console"` (or stays as a thin alias — decide during implementation; console pages need the admin gate + console chrome).
- Enable/disable (P-058) hooks in for free: the dispatcher checks `plugin_state` once, disabled plugin's pages 404 uniformly.

## What stays on `routes`

File routes keep pages that need real routing features: type-safe params, search-schema validation, nested layouts/loader inheritance, per-route code splitting conventions. `pages` is for self-contained pages and anything runtime/DB-driven. Rule of thumb: needs params/search/layout → `routes`; standalone page or dynamic path → `pages`.

## Consumers

- **Custom pages plugin** (P-058 escape hatch): DB rows → `pages` descriptors generated at request time — sidebar item + page, no code. This is the primary motivator.
- Simple plugin pages that don't justify route files (marketing sub-pages, status pages, embed hosts).
- `plugins/manager` itself could dogfood `pages` for `/platform/plugins` once `context: "console"` lands — or keep `panel.pages`, same machinery.

## Open questions

- One root splat vs per-section splats (`/platform/$` exists; add `/$` catch-all?) — catch-all risks shadowing future file routes; per-section is safer but requires knowing sections ahead. Lean: explicit splat mounts per namespace (`/p/$`? or reuse path prefixes plugins declare).
- Path conflicts: two plugins claiming `/about` — first in topo order wins + dev-time warning, or hard error at discovery?
- Does `pages` support server `loader` returning serializable data only (server fn boundary) — same constraint as panel.pages today.
- i18n: page paths are literal strings; localized routes (`/pt-br/about`) out of scope for v1.

## Non-goals

- Runtime plugin code (still bundle-only).
- Nested layouts / shared loaders across `pages` entries.
- Replacing `routes` for existing domain pages.
