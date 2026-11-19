+++
id = "P-104"
title = "List plugin-contributed pages in /platform/pages admin table"
status = "done"
created = "2026-11-19"
owner = "devin"
priority = "med"
done = "2026-11-19"
+++

## Problem

`/platform/pages` only lists `custom_pages` DB rows (`CustomPages.adminList()`). Plugin-contributed pages (`pages`/`panel.pages` descriptors) live in the compile-time `collectPages` registry (`appPages`/`portalPages`, each entry tagged `pluginId`) and are dispatched by splat routes — they never appear in the admin table, and there is no "provided by" column.

## Proposal

- `pages.panel` loader additionally returns registry entries (`path`, `pluginId`, `context`), filtered by enabled plugins (`enabledIdSet`/`entryEnabled`).
- `PagesAdminPage` renders a unified row list with a new "Provided by" column: plugin-id badge for registry rows, "Custom" for DB rows.
- Plugin rows are read-only (no edit/delete/external-view actions); wildcard `/*` descriptors shown as their declared path.
- New i18n keys in `pagesMessages` (`colProvided`, labels) + baseline regen.

## Touch points

`plugins/pages` (index.ts loader, platform-page.tsx narrowing, views/PagesAdminPage.tsx), `apps/web` registry plumbing to expose entries to the loader, i18n catalogs, tests.
