+++
id = "P-082"
title = "Theme plugin: white-label tokens, brand assets, accessible palettes"
status = "in-progress"
created = "2026-11-15"
priority = "med"
links = ["T-028", "P-053", "P-060", "P-070", "P-081"]
+++

# Theme plugin — white-label tokens, brand assets, accessible palettes

## Problem

The platform is single-brand: `@ktbhealth/ui` Tailwind v4 tokens are baked
into `globals.css` (`:root`/`.dark` CSS vars), the favicon/meta are static
in `apps/web/src/routes/__root.tsx`, and there is no per-tenant or
per-portal-instance branding. White-label (a clinic's logo, palette, app
name on its own portal) has no seam today.

## Proposal

A `plugins/theme` plugin (`theme.brand`) that turns the platform
white-label through the existing plugin contract — no kernel changes.

### Contribution kinds (generic channel, P-070)

New registered kinds consumed via `collectKindContributions`:

- `theme.tokens` — CSS custom-property overrides keyed by the exact var
  names `globals.css` already consumes (`--primary`, `--ring`,
  `--background`, `--sidebar`, …, plus `--ktb-*` aliases). Because every
  `@ktbhealth/ui` component styles off these vars, one override block
  re-skins the whole product — app shell, portal chrome, and any plugin
  view — with zero component changes.
- `theme.brand` — brand assets: logo (light/dark variants), wordmark,
  favicon (svg/png/ico), app name, og image. Consumed by the shells.

### Resolution order

`portal instance theme → tenant theme → platform default → ui tokens`.

- Themes are rows in a contributed `theme_definitions` table (fields:
  scope kind + id, token overrides as JSON, asset refs, enabled flag,
  version). Per-tenant theming rides the existing `tenant` scoping
  (P-053); per-portal-instance theming uses the `portal` field already on
  chrome contributions (P-081) — a clinic's doctor portal and its
  patient-facing portal can carry different brands.
- Theme resolution is PER-SHELL (the portal instance id isn't knowable
  in the root loader): the app shell resolves session → tenant →
  platform in the root loader; each portal shell (`platform.tsx`,
  `portal.$instance.$.tsx`) resolves instance → tenant → platform in
  its own loader and emits its own `head()` favicon/title (`head` ctx
  carries `loaderData`). Exactly one `<style>` emitter per page.
- Injections per shell: a `<style>` block of var overrides scoped to
  `:root`/`.dark`, `<link rel="icon">` + apple-touch-icon, and localized
  title/og meta from `theme.brand.appName`.
- Brand slots: `app.header.brand` already exists (PL-005) and renders in
  `__root__.tsx`; add `portal.brand` for the portal shells —
  `PortalShell`'s `SidebarHeader` currently hardcodes the "KTB"
  wordmark and gets a `brand` slot instead. The theme plugin contributes
  `kind: "component"` chrome into both regions, `portal`-scoped.

### Why CSS vars (confirmed mechanism)

`@ktbhealth/ui`'s `globals.css` uses Tailwind v4 `@theme inline`, mapping
`--color-*` → `var(--*)` — every utility (`bg-primary`, `text-ring`, …)
resolves the custom property at runtime. Overriding `--primary` &
friends on `:root`/`.dark` re-skins all components with zero build-time
per-tenant CSS. This is the shadcn/Tailwind-v4 model; `@ktb/*` aliases
(`--ktb-*` from Terrazzo) are overridden the same way for consumers
that read them directly.

### SSR / rendering guarantees

- **No flash, no FOUC.** Brand scope comes from the request (session →
  tenant/portal host), resolved server-side in each shell's loader —
  the `<style>` block, favicon links, and title ship in the initial
  HTML. No client-side theme detection script, no hydration-time swap.
- **Brand ≠ color scheme.** The plugin emits BOTH `:root` and `.dark`
  override blocks derived from the same seeds; `.dark` remains the
  user's light/dark preference, orthogonal to brand.
- **No layout shift.** The brand slot reserves a fixed aspect-ratio box
  for logo/wordmark (sizing via `[&_svg]`/aspect utilities, never
  intrinsic image size); favicons never affect layout. Fonts ARE
  themeable (see Typography) but ship with a metric-matched fallback so
  the swap can't shift layout.

### Accessibility — automatic, not advisory

"Accessible" is enforced by construction, not by reviewer discipline:

- **Seed colors, not pairs.** Theme authors supply seed hues (brand
  color, surface preference); the plugin derives every `*-foreground`
  pairing by computing WCAG contrast in OKLCH and clamping lightness
  until AA is met (4.5:1 body, 3:1 large/UI). Authors cannot submit an
  inaccessible fg/bg pair because fg is never an input.
- **Focus ring guaranteed.** `--ring` is derived from the primary with a
  contrast floor against both background and card — keyboard focus stays
  visible on any brand palette.
- **Dark mode paired.** A theme declares light+dark seed sets; the
  `.dark` var block is generated, not authored, so the two modes can't
  drift apart.
- **Validation at write time.** The config/table schema rejects
  out-of-gamut colors and sub-AA results that can't be auto-corrected
  (e.g. `destructive` keeps its semantic hue but gets a compliant
  foreground). The platform admin UI shows a live contrast report per
  pairing.
- **Honest limits.** Auto-derivation covers color contrast only; themes
  can't break semantics because components never consume raw hex —
  they consume the token layer.

### Typography — fonts are themeable

`--font-sans`/`--font-heading`/`--font-mono` (currently `--font-geist*`)
join the overridable token set. Two delivery modes, chosen per theme:

- **Self-hosted (default).** The asset job downloads the woff2 files
  (from Google Fonts or a user upload) into `THEME_ASSETS_BUCKET` and
  the shell emits `@font-face` + a **metric-matched fallback**
  (`size-adjust`/`ascent-override` etc.) so the swap can't cause CLS.
  Private, fast, and consistent with the on-prem/self-hosting
  positioning (T-082) — no visitor requests leave our origin.
- **Google Fonts `<link>` (opt-in).** Convenience mode for tenants who
  don't mind the external request; the trade-off (IP leak to Google,
  on-prem conflict) is explicit in the admin UI, not hidden.

### Legacy brand retirement

Today's brand is hardcoded host chrome, not a theme: the "KTB"
wordmark + instance badge in `PortalShell`, the `app.header.brand`
contribution in `plugins/site`, static meta/favicon in `__root__`. The
**platform-default theme row** subsumes all of it — once themes ship,
a final phase removes the hardcoded fallbacks so brand lives in
exactly one place (theme data).

### Admin surface

- `panel.pages` entry `/platform/theme`: pick scope (platform default /
  tenant / portal instance), edit seed colors with a live preview
  iframe, see the contrast report, publish.
- **Asset uploads are async jobs, never request-path work.** Upload →
  R2 (`THEME_ASSETS_BUCKET` binding, same shape as `AUDIO_BUCKET`) →
  enqueue → worker validates the image and derives variants (favicon
  16/32/180, resized logo svg/png/webp/avif, optional dominant-color
  extraction to *suggest* accessible seeds) → writes asset refs on
  `theme_definitions`. Port/adapter at the edge per ADR-0001; progress
  surfaces in the jobs console feed like audio jobs.
- **Theme switch in the platform header.** A `headerAction` chrome
  contribution into `portal.header.actions` with
  `optionalDeps: [platformHeaderPlugin.id]` — the exact graceful-
  degradation contract `jobs.header` already uses: renders only when
  the `platform.header` marker plugin is registered. Popover lists
  `theme_definitions` by scope for admin preview/switch.

### Non-goals / invariants

- No per-component overrides — if a clinic needs a layout change that's
  a plugin/page contribution, not theme.
- `@ktbhealth/ui` stays the token source of truth; the theme plugin only
  emits var overrides at runtime. No vendoring, no fork.
- Token emission is a static `<style>` tag (no runtime CSS-in-JS), so it
  works in workerd SSR and adds ~1KB.

## Alternatives considered

- **Runtime CSS-in-JS / style injection per component** — rejected:
  breaks SSR determinism, fights Tailwind v4's compile-time model, and
  bypasses the token layer that makes components themeable for free.
- **Theme as plugin `config` JSON only** — rejected: white-label needs
  per-tenant/per-instance rows and asset storage; config is
  per-plugin-install, not queryable per request scope.
- **Fork `globals.css` per deployment** — rejected: one build serves all
  tenants; per-deploy branding can't do portal-instance scoping.

## Links

- P-081 (portal plugin — per-instance scoping), P-060 (chrome regions),
  P-070 (generic contributions), P-053 (tenant scoping)
- T-028 (branding assets — favicon/logo source material)
- ui repo: `packages/tailwind/src/globals.css` token surface;
  `packages/tailwind/src/generated/tokens.css` (Terrazzo `--ktb-*`)
