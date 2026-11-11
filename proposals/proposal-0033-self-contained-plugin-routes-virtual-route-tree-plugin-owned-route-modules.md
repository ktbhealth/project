+++
id = "P-033"
title = "Self-contained plugin routes: virtual route tree + plugin-owned route modules"
status = "done"
created = "2026-11-09"
owner = "devin"
priority = "medium"
epic = "platform"
+++

## Context

Today a plugin that wants app routes still needs host wiring: the
plugin declares `routes: [route("consultations", [...])]` (virtual
nodes pointing at files) but the actual route modules live in
`apps/web/src/routes/**` because they wire plugin views to host server
functions (`#/lib/fns.ts`), which arch lint forbids plugins from
importing. Console pages are worse: `/platform/*` is a `physical()`
mount, so every console page is a file in the host's routes dir.

Goal: a plugin should be droppable — routes, nav, panel, tables,
services — with zero files added under `apps/web/src/routes/`.

## What the toolchain already supports (verified in
@tanstack/router-generator source)

- **Virtual (file-less) route nodes**: `route(path, children)` with no
  file produces an `isVirtual` node; the generator emits
  `createFileRoute('path')()` inline in `routeTree.gen.ts`. No route
  module required.
- **Route pieces**: `.lazy.tsx`, `.component.tsx`, `.loader.ts` files
  discovered by `physical()` directory scans attach to routes by path
  (`lazyRouteComponent(() => import(...))`). Piece files can live
  anywhere a `physical()` directory resolves — `resolve()` normalizes
  `..`, so directories outside `routes/` are mechanically possible
  (needs a spike to confirm watch/vite tolerate it).

## Blockers to solve

1. **Critical route options are unavailable on virtual routes.**
   Pieces cover only component/error/pending/notFound/loader/lazy.
   `beforeLoad`, `validateSearch`, `context` require a real route
   module. Our admin gate is a `beforeLoad` — guarded routes can't be
   file-less today. Options: (a) accept the limitation for simple
   routes; (b) upstream feature request to TanStack; (c) keep a thin
   host module only for guarded routes.

2. **Server-fn boundary.** Route modules need data. `createServerFn`
   is importable anywhere, but handlers need the app `ManagedRuntime`
   + session — currently reached via `apps/web/src/lib/platform.ts`
   (the deliberate `cloudflare:workers` env seam). Proposal: an
   injected-runtime package (e.g. `@ktb/fns` or extend app-host) —
   `setRuntime()`/`getRuntime()` module registry populated once by the
   composition root; plugins define their own server fns against it.
   `getRequestHeaders`/session helpers are already package-importable.
   This deliberately relaxes "plugins never touch runtime/env" — the
   env read stays host-side behind the registry.

3. **physical() outside routes/** — spike: point a physical dir at
   `../../plugins/<x>/src/routes` and verify generator imports, vite
   resolution, HMR/watch, and routeTree.gen.ts output paths.

## Phases

- **P1 — spike**: virtual route + `.lazy.tsx` piece in a plugin package
  mounted via `physical()` escape; verify build/dev/typegen.
- **P2 — runtime package**: extract fn helpers + runtime registry;
  migrate one plugin (users/groups — simplest, admin-only) to
  self-contained routes; measure DX.
- **P3 — console parity**: `/platform` subtree becomes plugin-declared
  virtual routes where possible; guarded routes keep thin host modules
  or wait on upstream critical-options support.
- **P4 — cleanup**: remove per-plugin entries from `apps/web/src/routes.ts`
  composition; arch lint updated to allow the new package boundary.

## Non-goals

- No runtime/dynamic plugin loading (workerd constraint stands).
- No change to the plugin contract fields themselves.
- Not blocking on TanStack upstream — ship what the current
  generator supports.

## Open questions

- Does `physical()` outside `routesDirectory` break routeTree.gen.ts
  import paths or vite fs.allow?
- Should critical-options routes keep a generated host stub, or do we
  press for upstream support first?
- Does the runtime registry belong in app-host or a new
  `@ktb/server-fns` package?
