+++
id = "P-026"
title = "Admin shell & authorization: /admin route group, admin role, dense system-console layout"
status = "open"
created = "2026-11-08"
priority = "high"
labels = ["admin"]
epic = "admin"
+++

## Idea

The `/admin` shell: route group, layout, sidebar nav, and the
authorization gate. An `admin` role in the authz package — checked
server-side on every admin route/handler, with the UI hiding nav items
but never relying on hidden UI for security.

System-alike shell: left sidebar (Entities / Ops / Billing / Audit),
top bar with env badge + current admin, dense table defaults, dark
theme optional. Reuses `packages/ui` primitives where they fit; admin
may need its own denser table component.

## Likely scope

- `apps/web` route group `/admin/*` behind the role guard
- `admin` role assignment: env allowlist initially (e.g.
  `ADMIN_EMAILS`), DB-backed later
- Shared admin layout: nav, page header, empty/loading/error states
- Denser table + status-badge primitives in `packages/ui` (or
  `packages/admin-ui` if divergence is large)

## Why it matters

Everything else in the admin epic mounts into this shell. The authz
gate is the security-critical piece — get it right once.

## Positioning cautions

- Role check on the server for every admin endpoint; client-side
  hiding is cosmetic only.
- Keep the shell boring: it should feel like an internal tool, not a
  product surface.

## Depends on

- P-025 (umbrella), authz package
