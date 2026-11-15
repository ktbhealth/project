+++
id = "P-082"
title = "Theme plugin: white-label tokens, brand assets, accessible palettes"
status = "open"
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
- The host resolves the active theme in the root loader (alongside
  `sessionFn`/`pluginStatesFn`) and injects it as:
  1. a `<style>` block of var overrides scoped to `:root`/`.dark`,
  2. `<link rel="icon">` + apple-touch-icon in `head()`,
  3. localized title/og meta from `theme.brand.appName`.
- New shell regions `app.brand` / `portal.brand` (declared alongside the
  existing `app.header.nav`, `portal.sidebar`) render the logo/wordmark;
  the theme plugin contributes `kind: "component"` chrome into them,
  filtered by its `portal` scope.

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

### Admin surface

- `panel.pages` entry `/platform/theme`: pick scope (platform default /
  tenant / portal instance), edit seed colors with a live preview
  iframe, upload logo + favicon (via `storage-dataset` or a dedicated
  asset table + R2 binding), see the contrast report, publish.
- `chrome` contribution `kind: "headerAction"` for a quick "preview as
  tenant" toggle during editing.

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
