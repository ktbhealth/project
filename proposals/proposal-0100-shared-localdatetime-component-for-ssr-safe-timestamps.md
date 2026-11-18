+++
id = "P-100"
title = "Shared <LocalDateTime> component for SSR-safe timestamps"
status = "in-progress"
created = "2026-11-18"
priority = "med"
labels = ["CI"]
+++

## Problem

`Date.prototype.toLocaleString()` resolves in the runtime timezone. During SSR the workerd worker renders in UTC while the browser renders in the user's timezone — whenever a table has rows, server and client emit different text → React hydration mismatch → console errors + client re-render.

First surfaced by the route crawl (PR #185 fixed 4 pages with a local `useSyncExternalStore` gate). ~10 views still carry the pattern and will error once their tables have data:

- plugins/platform: AdminConsultationsPage, AdminTenantsPage, OpsPage, MachinesPage, VaultSecretsPanel, AdminRbacPage, DocumentsAdminPage
- plugins/views: view-utils.ts + detail pages
- portal detail pages

## Proposal

1. Add a shared `<LocalDateTime>` (name TBD by implementer) component/hook — likely in `@ktb/ui` or a shared views package — that renders an empty/stable placeholder on the server and the localized timestamp on the client (useSyncExternalStore pattern already proven in #185).
2. Refactor the ~10 affected views to use it.
3. Keep formatting options (dateStyle/timeStyle/locale) flexible enough for current usages.

## Acceptance

- No hydration mismatch on any page with timestamped rows (verified via the /tmp/crawl.mjs playwright crawler).
- 100% coverage gate still passes; tests updated where snapshots/assertions cover the markup.
