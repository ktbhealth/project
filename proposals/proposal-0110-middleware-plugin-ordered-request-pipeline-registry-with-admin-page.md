+++
id = "P-110"
title = "Middleware plugin: ordered request-pipeline registry with admin page"
status = "open"
created = "2026-11-20"
priority = "med"
links = ["P-101", "P-106"]
+++

## Vision

A `middleware` platform plugin: an ordered, plugin-contributed request pipeline — auth, tenant resolution, rate-limit, captcha, audit, locale — where each stage is a registered middleware applied by route/context rules. Plus an admin page (`/platform/middleware`) listing registered stages, their order, and where they apply.

## Why

- Guards are currently scattered: `PlatformAdmin.requireAdmin` inside services, `guard` callbacks on page descriptors, `portalAdmits` audience checks, authz checks in handlers. There's no single place that answers "what runs on this request, in what order?"
- Auth is the prime consumer: session → principal → tenant → capability checks are exactly a middleware chain; making them explicit enables per-route policy (e.g. "this custom page requires auth", "this API is public") without code.
- Complements P-106 routes: middleware binds to route keys/contexts, not raw paths.
- Admin visibility: which stages are active, ordering, and per-route overrides — currently impossible to inspect.

## Shape (proposal)

- Kernel contribution kind `middleware` (or plugin manifest field): `{ id, order, appliesTo(path|routeKey|context), handler }` — handler wraps the request/next chain.
- Built-in stages first: principal resolution, tenant, platform-admin gate (absorbs `PlatformAdmin.requireAdmin` route-level use), locale.
- Plugin-contributed: rbac grants, captcha, rate-limit, audit logging.
- Admin page: table of stages (id, order, scope, source plugin, enabled) + per-route effective-chain view.
- Careful with Effect service boundaries: middleware runs at the app-host edge (composition root), services stay pure — same adapter-edge rule as arch lint.

## Open questions

- Pipeline model: onion (wrap next) vs linear stages with early-return — Effect idioms favor a scoped/middleware-free approach; evaluate.
- Does this replace descriptor `guard` callbacks or compose with them? (Recommend: guards become middleware-generated; descriptor guard stays as escape hatch.)
- Interaction with portal audience model (`portalAdmits`) — middleware vs portal config.
