+++
id = "P-062"
title = "Capability packages as plugins: auth/tenant/rbac audit — plugins add capabilities + plugin-plugin deps"
status = "in-progress"
created = "2026-11-11"
priority = "med"
labels = ["plugins"]
links = ["T-087", "P-058"]
epic = "platform"
+++

## Context

Refined boundary (AGENTS.md): `plugins/` = registry citizens that contribute tables/services/routes/nav/chrome/pages/panel AND can add capabilities other plugins consume via `requiredDeps`/`optionalDeps` + CQRS + EventBus. `packages/` = infra BELOW the plugin system (ports/adapters/drivers/env).

Audit of `packages/*` (services/tables/routes scan) shows most are correctly infra — but several are domain *capabilities* wearing infra clothing. User pushback: plugins should add capabilities and depend on plugins — they already can; the question is which packages should become plugin citizens.

## Classification

**Stay infra (ports/adapters/drivers — correctly placed):**
`db`, `db-sqlite`, `kernel`, `driver-framework`, `app-host`, `server-fns`, `authz` (pure port), `auth-better` (better-auth adapter), `bot`+`bot-recall` (port+driver), `captcha`+`captcha-stub`+`captcha-turnstile` (port+drivers), `transcription`+`-webspeech`+`-whisper` (ports/drivers), `anamnesis`+`anamnesis-llm` (port+driver), `interop`+`-fhir`+`-openehr` (ports/adapters), `export-pdf`, `tenant` (port: TenantResolver/TenantDirectory — domain bits already wrapped by plugins/tenant on main).

**Capability packages → plugin citizens (or thin wrappers, rbac/tenant pattern):**

- **`packages/auth`** — the flagship candidate. It owns tables (user/session/account/verification — now in @ktb/db), a service (`Auth.getSession`), API surface (`/api/auth/*` catch-all currently a route in apps/web), and UI-adjacent concerns (login page lives in apps/web). As `plugins/auth`: services (session resolution), routes (`/api/auth/*` + maybe `/login`), possibly chrome (session menu already exists as plugins/session — merge?). Everything authz depends on it → `requiredDeps` edge from consumers. Big-bang risk: auth is foundational; everything breaks if its plugin is disabled — that's FINE, disabling auth should probably be impossible (mark non-disableable / system plugin).
- **`packages/rbac`** — domain lib; `plugins/rbac` wrapper already exists on main (P-054). Pattern proven.
- **`plugins/audit`, `plugins/tenant`** — already landed on main via the wrapper pattern. Confirms direction.

## Proposal

1. Codify: a package becomes a plugin when it owns registry-worthy contributions (tables+services+routes+UI). Ports/adapters/drivers stay packages forever.
2. Pilot: `packages/auth` → `plugins/auth` (or `plugins/auth-better` wrapper). Move `/api/auth/*` route module + login page route into the plugin; session service stays a port consumed via layer. Mark auth plugin as non-disableable in plugin_state (system plugin — P-058 manager should show it locked).
3. `packages/tenant` domain services → fold into `plugins/tenant` if they're registry-worthy; keep `TenantResolver` port in packages.
4. Follow-up: audit `packages/anamnesis` — it's a domain engine (LLM anamnesis) with a service; could be plugins/anamnesis if it gains routes/UI.

## Open questions

- Can a plugin be non-disableable? (auth, platform, manager itself?) — plugin_state needs a `locked`/`system` flag or manager UI hardcodes protected ids.
- Does making auth a plugin break `createApp`'s manual Auth wiring (it's deliberately NOT a driver — see app-host comments)? The plugin would wrap the same makeAuth instance, not replace it.
- Login page (`apps/web/src/routes/login.tsx`) → plugin route? It uses authClient directly; movable.
