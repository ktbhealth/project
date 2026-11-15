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
- `apps/web/src/lib/registry.ts`: add `app.brand` + `portal.brand` to `KNOWN_CHROME_REGIONS`.
- `__root__.tsx` + portal shell: declare brand regions; render resolved brand (logo/wordmark) in a fixed-aspect slot (CLS guard — never intrinsic image size).

## Phase 2 — `plugins/theme` domain core

- `plugins/theme/src/index.ts` (+ client.ts parity): tables plugin `theme_definitions` (scope: platform|tenant|portal-instance + scope id, seeds JSON, derived tokens JSON, asset refs, enabled, version) and `theme_assets` (r2 key, kind, variants JSON, status).
- `Theme` Effect service behind a port: `resolve(scope)` → `{ tokens: {light, dark}, brand, appName }` with fallback chain portal-instance → tenant → platform → null (ui defaults). Authz: reads open to session; writes PlatformAdmin (+ tenant admin later).
- Token engine (`src/palette.ts`): seeds → full shadcn var set. OKLCH math, WCAG contrast, AA clamp for every `*-foreground`, `--ring` contrast floor vs background+card, `.dark` block derived from same seeds. Pure functions → trivially testable to 100%.

## Phase 3 — Host injection (SSR guarantees)

- Root loader: resolve active theme from request scope (session → tenant → portal host) alongside `sessionFn`/`pluginStatesFn`.
- `head()`: favicon links (svg/png/apple-touch) + localized title/og from `theme.brand.appName`.
- Render `<style>` block with `:root`/`.dark` var overrides in `RootDocument` — initial HTML, no client detection, no flash. Brand ≠ color-scheme: `.dark` stays user pref.

## Phase 4 — Async asset pipeline

- `wrangler.jsonc`/`wrangler.dev.jsonc`: `THEME_ASSETS_BUCKET` R2 binding (mirror `AUDIO_BUCKET`).
- Upload api-route in `plugins/theme/src/api-routes/`: authz → R2 put → enqueue job message (same shape as consultations audio upload).
- Queue worker/handler: validate image → derive variants (favicon 16/32/180, logo png/webp/avif, optional dominant-color → suggested seeds) → update `theme_assets` + `theme_definitions` refs. No sync image work in request path.
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
