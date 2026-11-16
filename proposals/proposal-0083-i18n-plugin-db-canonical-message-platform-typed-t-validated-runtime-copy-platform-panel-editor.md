+++
id = "P-083"
title = "i18n plugin: DB-canonical message platform — typed t(), validated runtime copy, platform panel editor"
status = "open"
created = "2026-11-15"
priority = "med"
links = ["T-082", "P-065", "P-070", "P-081", "P-082", "ADR-0003", "PL-008"]
+++

# i18n plugin — DB-canonical message platform

## Problem

Copy is compile-time data in two disjoint systems:

- **Host chrome** — Paraglide JS compiles `apps/web/messages/{en,es,pt-BR}.json`
  into typed functions (`m.nav_patients()`). No runtime key lookup, no
  DB, locale set fixed at build time. Paraglide was built for this
  compile-time model — it is not made for runtime-editable messages.
- **Plugin copy** — each plugin hand-rolls a typed catalog in
  `plugins/*/src/i18n.ts` (plugins cannot import Paraglide output —
  arch lint). The host injects `locale` via `PageContext`/
  `PanelPageContext`/`ChromeProps`.
- **Plugin chrome labels** — English literals in manifests, localized
  through hand-maintained maps in `apps/web/src/lib/i18n.ts`
  (`NAV_LABELS`, `PANEL_LABELS`, …).

Every wording fix is a code change + deploy. T-082 shows copy decisions
are contested; P-065 makes locale-native quality the wedge; P-082
white-label and P-081 portals both want per-scope wording. Messages
should be **data**, editable at runtime, validated, and reviewable —
not compiled artifacts.

## Proposal

An i18n system plugin (`plugins/i18n.*` pieces + a `@ktb/i18n` port,
following the `packages/rbac` + `plugins/rbac` precedent) that makes
the database the canonical store for ALL UI copy and retires both
Paraglide and the per-plugin `i18n.ts` catalogs.

### Data model

- `i18n_namespaces` — namespace registry: id, owning plugin, key manifest
  hash, default locale.
- `i18n_messages` — the canonical store: `namespace.key × locale ×
  scope → { value, status: draft|published, params signature }`.
  Scope = `global | tenant | portal instance` (P-082 theme precedent).
  Rows are full messages, not deltas — the DB is the source of truth.
- Seed bundle: a JSON snapshot generated at build (`gen:i18n-seed`)
  from the last DB export, shipped with the worker so the app renders
  on cold start / empty DB / DB outage. DB rows override seed per key.

### Codegen keeps type safety

- `gen:i18n-types` emits a `.d.ts` from the message manifest (keys +
  declared params per namespace) → `t("nav.patients", { name })` stays
  fully typed with param checking. CI regenerates and diffs — drift
  fails the build. Compile-time checking becomes CI-time checking.
- `gen:i18n-manifest` walks plugin contributions + the seed bundle to
  produce the key/param manifest the panel and validator share.

### Resolution service

`@ktb/i18n` port: `I18n.t(key, params)` resolves per request scope:

`portal instance → tenant → global` scope chain, then locale fallback
(`pt-BR → en` etc. per `i18n.locale` declarations).

- Per request, load applicable published rows once into an immutable
  `Map`; `t()` is a pure lookup + render. Version-stamped per-isolate
  cache, invalidated on write — never query per `t()` call (T-063
  bundle/request budget).
- **Message format**: ICU MessageFormat subset — `{param}`
  interpolation plus `plural`/`select` where needed. v1 implements a
  ~100-line safe subset in-house (no formatjs — worker bundle cost);
  full ICU is a later option.
- **Validation** (shared parser, enforced on write AND in CI):
  params must match the key's declared signature exactly; no markup;
  length caps; plural/select category coverage per locale.

### Contribution kinds (generic channel, P-070)

- `i18n.locale` — `{ code, endonym, fallback?, enabled }`. Runtime
  locale enable/disable + picker metadata. New locales are data rows —
  **no rebuild required** (the whole point of data-first).
- `i18n.messages` — optional contributed seed catalogs for plugins that
  ship copy with the package; imported into `i18n_messages` on sync.
  Existing `plugins/*/src/i18n.ts` files become seed sources, then
  retire.

### Migration (full — this is the point)

- `apps/web/messages/*.json` → one-time import into `i18n_messages`
  (en/es/pt-BR rows, global scope). Paraglide vite plugin removed;
  `m.*` call sites migrate to `t()` (`lib/i18n.ts` chrome first, codemod
  for route-level usage).
- Plugin `i18n.ts` catalogs → seed import via `I18nCatalog` shim during
  transition, then deleted. Host injects locale/catalog through
  `@ktb/i18n` instead of per-plugin props over time.
- English manifest literals in chrome contributions resolve through
  `t()` with the literal as fallback during transition.

### Platform panel

`/platform/i18n` (PanelPlugin + chrome sidebar entry):

- **Matrix view**: rows = keys grouped by namespace, columns = locales
  × scopes. Badges: draft, overridden-from-seed, missing translation,
  param mismatch. This is the primary editing surface.
- Cell editing with live preview rendered with sample params
  (`{name}` → "Maria"), pseudo-locale toggle for truncation checks,
  base-vs-effective diff, reset-to-seed per cell/namespace.
- Bulk ops: copy locale column, fill-missing from fallback locale.
- Import/export JSON per namespace×locale with a validation report —
  nothing partially applied.
- Filters: draft/published, missing-in-locale, has-override, fuzzy
  full-text search.
- Optional scheduled JSON export committed to the repo for diffable
  copy history if git-backed review is wanted alongside draft→publish.

### Access control

Permission-scoped via rbac, NOT admin-only: `i18n.messages.edit`
granted per scope — an actor can edit tenant/portal rows only where
authorized; global rows stay platform-level. Read is broader (matrix
visible to console users); write checks happen inside the service
layer like every other panel loader. Permission scope and data scope
line up by construction.

## Why this shape

- **Data-first is the product fit**: multi-tenant, white-label, and
  locale-native (P-065) all want runtime-editable, per-scope copy.
  Compile-time catalogs were the wrong tool; Paraglide is retired, not
  wrapped.
- **Safety is preserved, relocated**: typed `t()` + param signatures
  via codegen, CI drift check, write-path validation, draft→publish +
  audit (P-080) replace PR review for copy. T-082 decisions land as
  data with full history.
- **System, not feature**: `i18n.tables`, `i18n.services`,
  `i18n.panel`, `i18n.chrome` — split by concern (one typed
  contribution kind per plugin object).
- **Precedents**: generic kinds (P-070), portal scoping (P-081),
  DB-backed definitions (P-082), rbac-scoped panel writes.

## Alternatives considered

- **Compile-time libs** (Paraglide, Lingui, typesafe-i18n): same
  model mismatch that motivated this proposal — messages are data,
  not build artifacts. Out.
- **i18next**: closest runtime-resolver model, but ~18kB, suffix-plural
  format (ICU needs a plugin), weakest typing; scoped resolution would
  still be custom. Rejected.
- **react-intl / FormatJS**: full ICU MF1 but ~15kB and the weakest
  typing of the set — a format engine to bolt on, not the framework.
  Its parser remains an option if full ICU is ever needed.
- **Tolgee**: the UX reference (in-context editing, translation
  states, TM, MT) but a whole Spring/Kotlin/Postgres service — wrong
  stack for a Workers monorepo. Steal UX ideas, don't adopt.
- **Native `Intl.MessageFormat`**: TC39 Stage 1, stalled — cannot be
  a dependency.
- **MF2 / `messageformat` lib**: successor format with real
  momentum (~340k weekly downloads) but early adoption; MF1 remains
  the translator/tooling standard. Revisit later.
- **Effect-ecosystem check**: no Effect-native i18n library fits.
  `@craft-ts/i18n-effect` has the right typed-key/Layer shape but is
  bound to the CraftTS framework with compile-time catalogs;
  KrakStack `localization` is locale-resolution middleware only;
  `@effuse/i18n` targets a different framework. Confirms the custom
  port — and Effect makes it natural: `I18n` as `Context.Tag`+`Layer`,
  scoped catalog load via `Effect.cached`, `Ref` version-stamp cache,
  Schema-based write validation.
- **Custom thin framework** (chosen): the novel surface — scoped
  resolution, DB-canonical store, codegen types, validation — has no
  off-the-shelf answer. Commodity parts stay native: `Intl.PluralRules`
  for plurals, `Intl.*` for number/date formatting, ~zero-cost React
  binding (locale switch reloads; `t()` is a sync lookup). The custom
  code is ~1–2k lines of pure, testable functions: schema, resolver,
  MF1-subset parser doubling as validator, codegen, seed pipeline.

## Phasing

- **P0** — `i18n_namespaces`/`i18n_messages` tables, `@ktb/i18n`
  service (resolve + validate + render), seed bundle pipeline,
  `gen:i18n-types`, panel matrix reader.
- **P1** — panel editing (draft→publish), import/export, missing-
  translation + param-parity reports; migrate ONE plugin catalog
  end-to-end as the reference.
- **P2** — migrate all plugin `i18n.ts` catalogs and chrome label
  maps; `i18n.locale` kind drives picker/locales; retire Paraglide
  for plugin copy.
- **P3** — host `messages/*.json` import, `m.*` → `t()` migration,
  remove paraglideVitePlugin; per-portal-instance forced/default
  locale.

## Risks / open questions

- **Review workflow**: copy changes leave the PR flow — draft→publish
  + audit must be disciplined; optional git export mitigates.
- **Type pipeline**: `gen:i18n-types` must run in CI before
  typecheck; ordering with `gen:route-types` needs care.
- **Seed drift**: seed bundle vs DB divergence on fresh envs — seed
  sync on deploy, CI check that seed ⊆ published messages.
- **Cold-start correctness**: empty-DB fallback must render complete
  UI; test the seed-only path explicitly.
- **ICU subset scope**: plural/select categories differ per locale —
  validator warns; full ICU deferred.
- **Bundle/request cost** (T-063): message maps cached per-isolate by
  version stamp; renderer must stay dependency-light.
- **100% coverage gate**: resolver, validator, and panel fns need
  full tests — keep `t()` resolution a pure function.

## Links

ADR-0003 (decision record), PL-008 (execution plan). P-065
(locale-native wedge), P-070 (generic kinds), P-081 (portal
instances), P-082 (scoped definitions precedent), T-082 (copy
decisions), P-080 (audit feed for edit history), T-063 (bundle
budget).
