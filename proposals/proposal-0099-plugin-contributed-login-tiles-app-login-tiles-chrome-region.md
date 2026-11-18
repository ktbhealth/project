+++
id = "P-099"
title = "Plugin-contributed login tiles (app.login.tiles chrome region)"
status = "open"
created = "2026-11-18"
owner = "devin"
priority = "med"
links = ["P-058"]
+++

## Problem

The `/login` role picker hardcodes all sign-in tiles in `apps/web`
(`ROLES`: to/key/icon/portal) and filters them via
`visibleLoginPortals` + `LOGIN_ROLE_PKGS`. A new role requires app-side
edits, and the package-dependency knowledge for each flow is duplicated
between `apps/web/src/lib/login-roles.ts` and the plugin route loaders
(`plugins/auth`, `plugins/secretaries`).

## Proposal

Re-use the chrome contribution machinery: add a `loginTile` content kind
(`{ to, labelKey, icon, requires?: ids[] }`) and a new region
`app.login.tiles`. Each role plugin contributes its own tile; the host
picker renders `collectChrome(enabledPlugins, "app.login.tiles")`
filtered by the P-058 live check (`entryEnabled`/`packageLive` + the
tile's `requires`). `login-roles.ts` and the `ROLES` table are deleted —
dependency knowledge lives with the plugin that owns it.

## Pieces

1. `@ktb/kernel`: `loginTile` ChromeContent kind + region in
   `KNOWN_CHROME_REGIONS`.
2. `auth.routes`, `secretaries.routes`, `patient-portal` (or app
   fallback for patient): contribute tiles.
3. `apps/web`: `/login` renders collected tiles; delete
   `login-roles.ts`.

## Open questions

- Label i18n: host `labels` injection vs plugin catalog `labelKey`.
- `requires` id space: registry entry ids resolved via `packageLive`.
