+++
id = "P-083"
title = "i18n plugin: runtime message overrides + platform panel editor over Paraglide/plugin catalogs"
status = "open"
created = "2026-11-15"
priority = "med"
links = ["T-082", "P-065", "P-070", "P-081", "P-082"]
+++

# i18n plugin — runtime message overrides + platform panel editor

## Problem

Copy is split across two disjoint systems with no runtime seam:

- **Host chrome** — Paraglide JS compiles `apps/web/messages/{en,es,pt-BR}.json`
  into typed functions (`m.nav_patients()`). Compile-time only: no dynamic
  key lookup, no DB overrides, locale set is fixed at build time.
- **Plugin copy** — each plugin hand-rolls a typed catalog in
  `plugins/*/src/i18n.ts` (plugins cannot import Paraglide output — arch
  lint). The host injects `locale` via `PageContext`/`PanelPageContext`/
  `ChromeProps`.
- **Plugin chrome labels** — nav/sidebar/footer strings are English
  literals in manifests, localized through hand-maintained lookup maps in
  `apps/web/src/lib/i18n.ts` (`NAV_LABELS`, `PANEL_LABELS`, …).

There is no way to fix a typo, adjust tone, or white-label wording
without a code change + deploy. T-082 already shows copy decisions are
contested; P-065 makes locale-native quality a core wedge. A platform
panel to view/edit effective messages — with base copy still in code —
is the missing ops surface.

## Proposal

An i18n system plugin (`plugins/i18n.*` pieces + a `@ktb/i18n` port,
following the `packages/rbac` + `plugins/rbac` precedent) that adds a
**runtime override layer** on top of existing catalogs — Paraglide stays
the compile-time base, plugin `i18n.ts` catalogs stay the typed base.
Overrides are deltas, never replacements of source copy.

### Contribution kinds (generic channel, P-070)

- `i18n.locale` — `{ code, endonym, fallback?, enabled }`. Governs which
  compiled locales are selectable and their display names at runtime.
  NOTE: adding a *new* locale still requires a rebuild (Paraglide
  compiles per-locale modules); this kind governs enable/disable +
  metadata, not locale authoring.
- `i18n.messages` — optional contributed catalogs for plugins that want
  data-driven copy without hand-rolled `i18n.ts` (escape hatch, not the
  default — typed catalogs keep compile-time completeness).

### Resolution service

`@ktb/i18n` port: `I18n.t(scope, key, params)` / `I18n.catalog(pluginId)`
resolves, per request scope:

`portal/tenant override → global override → plugin catalog / Paraglide fn`

- Overrides live in a contributed `i18n_overrides` table: namespaced key
  (`plugin.key` or Paraglide message id), locale, scope
  (global | tenant | portal instance — same scoping precedent as
  P-082 theme), value, updated-by/at. Per-portal-instance overrides give
  a clinic's pt-BR portal its own wording for free.
- Plugin catalogs plug in through a uniform `I18nCatalog` contract so
  `resolveMessages(catalog, locale)` feeds the same override lookup —
  `i18n.ts` files migrate incrementally, no big-bang.
- Host Paraglide messages become overridable via a small generated
  key→function map emitted alongside `src/paraglide` (the message ids
  are already stable identifiers); `lib/i18n.ts` wrappers consult the
  override store first. Overrides use Paraglide's message syntax
  (ICU MessageFormat params) so panel edits stay valid if exported back
  into `messages/*.json`.

### Platform panel

`/platform/i18n` page (PanelPlugin, platform portal instance):

- Namespace → key → locale matrix; edit override per locale with
  base-message preview and effective-message preview.
- Missing-translation report (keys lacking pt-BR/es) and ICU param-parity
  lint across locales.
- Export JSON per namespace/locale — the path back into code when a
  runtime edit should be promoted to source copy.
- Sidebar entry via `chrome` into `console.*` regions.

## Why this shape

- **Paraglide-compatible**: it remains the compile pipeline and typed
  base. The override layer sits at the service boundary, so we keep
  type safety, tree-shaking, and the 100% coverage gate's compile-time
  guarantees. Runtime catalogs replacing Paraglide entirely was
  considered and rejected — it loses all of that for no gain.
- **System, not feature**: like `platform.*`/`rbac`, it sits below
  feature plugins; ids follow `<domain>.<concern>` (`i18n.tables`,
  `i18n.services`, `i18n.panel`, `i18n.chrome` — split by concern since
  a plugin object carries one typed contribution kind).
- **Matches existing precedents**: generic `contributions[]` kinds
  (P-070), portal-instance scoping (P-081), DB-backed definitions with
  code fallback (P-082 theme).

## Phasing

- **P0** — `i18n_overrides` table, `@ktb/i18n` service, panel editor
  scoped to plugin catalogs only.
- **P1** — `I18nCatalog` contract; migrate plugin `i18n.ts` files
  incrementally; `i18n.locale` kind drives the picker/labels;
  missing-translation report.
- **P2** — Paraglide host messages editable via the key map; export
  workflow back to `messages/*.json`.
- **P3 (optional)** — per-portal-instance forced locale / default
  locale (clinic portal pinned to pt-BR).

## Risks / open questions

- ICU param parity between Paraglide syntax and DB overrides — needs a
  shared validator in `@ktb/i18n`, enforced in the panel and on write.
- Coverage gate: resolution service + panel fns need full tests; keep
  the override lookup a pure, easily-tested function.
- Workerd bundle/request cost (T-063): override maps must be cached
  per-isolate with invalidation on write, not re-queried per `t()` call.
- Copy governance: runtime edits can diverge from reviewed source copy —
  panel is platform-admin only; export-promote flow keeps code as the
  source of truth. T-082 decisions should land in `messages/*.json`
  first, then be overridden if ever needed.
- Locale *authoring* stays build-time; the proposal should say so
  explicitly so nobody expects runtime locale addition.

## Links

P-065 (locale-native wedge), P-070 (generic kinds), P-081 (portal
instances), P-082 (theme scoping precedent), T-082 (copy decisions).
