+++
id = "PL-006"
title = "Theme plugin (P-082): white-label tokens, brand assets, accessible palettes"
status = "open"
created = "2026-11-15"
priority = "med"
links = ["P-053", "P-060", "P-070", "P-081", "P-082"]
+++

Execution plan for P-082. Conventions: ports/adapters at the edge (ADR-0001), plugin contract single-concern fields + generic `contributions` (P-070), 100% vitest coverage gate.

## Phase 1 — Kernel kinds + shell brand slots

- `packages/kernel/src/kinds.ts`: register `theme.tokens` and `theme.brand` kind descriptors (meta schemas: token map = JSON object of CSS var → oklch/color string; brand = { logoLight, logoDark, wordmark, favicon, appName, ogImage } asset refs).
- `packages/kernel/src/chrome.ts`: `app.header.brand` ALREADY exists (PL-005 landed) — only add `portal.brand` to `KNOWN_CHROME_REGIONS`.
- App shell: `__root__.tsx` already renders `app.header.brand` entries — no slot work needed there.
- Portal shell: `plugins/portal/src/views/PortalShell.tsx` `SidebarHeader` hardcodes the "KTB" wordmark + instance badge — add a `brand` ReactNode prop rendered in that fixed slot (keep the instance badge host-owned); `platform.tsx` + `portal.$instance.$.tsx` collect `portal.brand` chrome per instance and pass it down. Fixed aspect box (CLS guard — never intrinsic image size).

## Phase 2 — `plugins/theme` domain core

- `plugins/theme/src/index.ts` (+ client.ts parity): tables plugin `theme_definitions` (scope: platform|tenant|portal-instance + scope id, seeds JSON, derived tokens JSON, asset refs, enabled, version) and `theme_assets` (r2 key, kind, variants JSON, status).
- `Theme` Effect service behind a port: `resolve(scope)` → `{ tokens: {light, dark}, brand, appName }` with fallback chain portal-instance → tenant → platform → null (ui defaults). Authz: reads open to session; writes PlatformAdmin (+ tenant admin later).
- Token engine (`src/palette.ts`): seeds → full shadcn var set. OKLCH math, WCAG contrast, AA clamp for every `*-foreground`, `--ring` contrast floor vs background+card, `.dark` block derived from same seeds. Pure functions → trivially testable to 100%.

## Phase 3 — Host injection (SSR guarantees)

Resolution is PER-SHELL, not root-global: the portal instance id isn't knowable in the root loader, and `head` ctx carries `loaderData` (verified in router-core types) so each shell's own `head()` can emit its theme's favicon/title.

- App shell (`__root__.tsx` loader): resolve theme for session scope (tenant → platform) alongside `sessionFn`/`pluginStatesFn`; `head()` emits favicon links + title/og from brand; `RootDocument` renders the `<style>` block (app chrome pages only — `isConsole` children path must NOT double-emit).
- Portal shells (`platform.tsx`, `portal.$instance.$.tsx` loaders): resolve instance → tenant → platform; each defines `head()` for favicon/title and passes resolved theme into `PortalShell`, which renders the `<style>` block + brand slot. Exactly one `<style>` emitter per page (a route is either app-chrome or portal).
- `<style>` contains BOTH `:root` and `.dark` override blocks from the same seeds; `.dark` stays the user pref (orthogonal to brand). No client detection, no flash — all server-rendered.

## Phase 4 — Async asset pipeline

- `wrangler.jsonc`/`wrangler.dev.jsonc`: `THEME_ASSETS_BUCKET` R2 binding (mirror `AUDIO_BUCKET`).
- Upload api-route in `plugins/theme/src/api-routes/`: authz → R2 put → enqueue job message (same shape as consultations audio upload: `AUDIO_QUEUE` producer pattern — see `consultations.$id.audio.ts`).
- Queue consumer: the `ktb-transcription` consumer lives in app lib (`audio-jobs.ts`) — add a `theme-asset` message kind dispatched to the theme pipeline, or a dedicated `ktb-theme-assets` queue (open decision below). Worker validates image → derives variants (favicon 16/32/180, logo png/webp/avif, optional dominant-color → suggested seeds) → updates `theme_assets` + `theme_definitions` refs. No sync image work in request path.
- Optional: surface theme-asset jobs in jobs console feed (jobs plugin consumes projections).

## Phase 5 — Admin UI + theme switch

- `panel.pages` `/platform/theme`: scope picker (platform/tenant/portal-instance), seed color editors, live preview (iframe `?theme=<id>` preview mode), contrast report per pairing, publish.
- `chrome` `headerAction` into `portal.header.actions`, `optionalDeps: [platformHeaderPlugin.id]` (jobs.header degradation contract): popover listing themes by scope for preview/switch.

## Phase 6 — Verify

- Unit: palette contrast/clamp edge cases, resolve() fallback chain, schema validation.
- Service/component tests per package scope (`cd plugins/theme && bunx vitest run`); `bun run check` green; SSR smoke: curl HTML contains `<style>` overrides + favicon links pre-hydration.

## Open decisions

- Reuse audio queue vs dedicated theme queue (lean: same queue infra, distinct message kind).
- `theme_definitions` scope modeling: polymorphic scope columns vs per-scope tables.
- Tenant-admin (non-platform) theme writes — v1 PlatformAdmin only?
- Preview mechanism: `?theme=` query override gated to PlatformAdmin.
