+++
id = "P-083"
title = "i18n plugin: DB-canonical message platform — typed t(), validated runtime copy, platform panel editor"
status = "done"
created = "2026-11-15"
priority = "high"
links = ["T-082", "P-065", "P-070", "P-081", "P-082", "ADR-0003", "PL-008"]
done = "2026-11-16"
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
  Scope = `global | tenant | portal instance` (P-082 theme
  precedent). Rows are full messages, not deltas — the DB is the
  source of truth. `value` is the materialized read model.
- `i18n_crdt_updates` — per-message Loro CRDT op log
  (`@loro-crdt/loro`): concurrent panel edits merge automatically;
  writes append ops and re-materialize `value` on the row, so reads
  (`t()`) never touch CRDT state — only the edit path does.
- `i18n_locales` — runtime locale rows: code, endonym, fallback chain,
  enabled flag. New locales are data, no rebuild.
- **Defaults are DB rows, not a bundle**: baseline copy is seeded into
  `i18n_messages` as `global`/`published` rows via migrations + the
  one-time import. The DB is both canonical store and fallback — no
  message JSON ships inside the worker bundle, and "empty DB" only
  exists pre-migration when the app isn't serving anyway.

### Codegen keeps type safety

- `gen:i18n-types` emits a `.d.ts` from the message manifest (keys +
  declared params per namespace) → `t("nav.patients", { name })` stays
  fully typed with param checking. CI regenerates and diffs — drift
  fails the build. Compile-time checking becomes CI-time checking.
- `gen:i18n-manifest` walks plugin `i18n.messages` contributions +
  baseline message files (`messages/*.json`, plugin catalogs during
  transition) + DB-declared namespaces → the key/param manifest the
  panel, validator, and CI drift check share.

### Resolution service

`@ktb/i18n` port: `I18n.t(key, params)` resolves per request scope:

`portal instance → tenant → global` scope chain, then locale fallback
(`pt-BR → en` etc. per `i18n.locale` declarations).

- Per request scope, load applicable published rows once into an
  immutable `Map`; `t()` is a pure lookup + render. **Caching**:
  per-isolate `Map` per (scope-chain × locale) behind a `Ref`
  version stamp, invalidated on write — the stamp itself is re-read at
  most every ~15s per isolate (and immediately after local writes), so
  `t()` never hits the DB (T-063 budget).
- **Chunking**: namespace is the unit everywhere — tables, manifest,
  generated types, panel tree, client payload (`patients.*`,
  `consultations.*`, `platform.*`, …).
- **Lazy loading**: SSR injects only the namespaces a route needs;
  the client fetches per-route namespace slices via server fn on
  navigation. Locale switch stays full-reload, so only the active
  locale × active namespaces ever ship — no per-locale chunk graph.
- **Message format**: ICU MessageFormat subset — `{param}`
  interpolation, formatter hints (`{amount, number}`,
  `{joined, list}`, `{when, date|datetime|relative}`) rendered via
  `Intl.*`, plus `plural`/`select` via `Intl.PluralRules`. Two message
  kinds: `text` (plain, no markup) and `rich` (sanitized HTML produced
  by the `@ktbhealth/ui` PlateJS `RichTextEditor` — paragraphs,
  h2/h3, bold/italic/underline, links, lists; rendered through the
  same component read-only). v1 implements a ~200-line safe subset
  in-house for `text` (no formatjs — worker bundle cost); full ICU is
  a later option. Rich HTML is sanitized on write AND on render
  (tag/attr allowlist — no scripts, handlers, or unknown tags).
- **Validation** (shared parser, enforced on write AND in CI):
  params must match the key's declared signature exactly; `text`
  messages reject markup, `rich` messages pass the HTML allowlist;
  length caps; plural/select category coverage per locale (against a
  CLDR-derived categories table baked into `@ktb/i18n` by one-time
  codegen — `Intl.PluralRules` can't enumerate categories; no
  per-locale override in v1). Missing keys resolve `scope → locale → default locale →`
  the literal `namespace.key` — the key is a visible sentinel, never a
  silent blank or English masquerading as localized; every
  fallback-to-literal is counted (metric + access log) so untranslated
  keys are measurable, and CI/panel reports flag non-default-locale
  gaps.

### Contribution kinds (generic channel, P-070)

- `i18n.locale` — `{ code, endonym, fallback?, enabled }`. Runtime
  locale enable/disable + picker metadata. New locales are data rows —
  **no rebuild required** (the whole point of data-first).
- `i18n.messages` — optional contributed baseline catalogs for plugins
  that ship copy with the package; imported into `i18n_messages` on
  sync. Existing `plugins/*/src/i18n.ts` files become baseline sources,
  then retire.

### Migration (full — this is the point)

- `apps/web/messages/*.json` → one-time import into `i18n_messages`
  (en/es/pt-BR rows, global scope). Paraglide vite plugin removed;
  `m.*` call sites migrate to `t()` (`lib/i18n.ts` chrome first, codemod
  for route-level usage).
- Plugin `i18n.ts` catalogs → baseline import via `I18nCatalog` shim
  during transition, then deleted. Host injects locale/catalog through
  `@ktb/i18n` instead of per-plugin props over time.
- English manifest literals in chrome contributions resolve through
  `t()` with the literal as fallback during transition.

### Platform panel

`/platform/i18n` (PanelPlugin + chrome sidebar entry):

- **Matrix view**: rows = keys grouped by namespace, columns = locales
  × scopes. Badges: draft, differs-from-baseline, missing translation,
  param mismatch. This is the primary editing surface.
- Cell editing with live preview: the message template renders via
  **Streamdown** with a custom plugin mapping `{param}` /
  `{param, hint}` tokens to styled chips showing live sample values
  (`{name}` → "Maria", `{amount, number}` → "€1,234.50"); clicking a
  chip edits its sample. Pseudo-locale toggle for truncation checks,
  base-vs-effective diff, reset-to-baseline per cell/namespace.
  `rich` messages edit through the `@ktbhealth/ui` PlateJS
  `RichTextEditor`; `text` messages through a plain input/textarea.
- Concurrent edits merge via **Loro CRDT** (`@loro-crdt/loro`) —
  per-message CRDT doc, op log in `i18n_crdt_updates`, materialized
  `value` re-written on the row; cursors/presence come free if we
  want them later. Every write hits the audit log.
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
  MF1-subset parser doubling as validator, codegen, baseline import.

## Phasing

- **P0** — `i18n_namespaces`/`i18n_messages`/`i18n_locales`/
  `i18n_crdt_updates` tables, `@ktb/i18n` service (resolve + validate
  + render + CRDT apply/materialize), baseline import pipeline,
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
- **Catalog growth**: namespace chunking + per-route lazy slices +
  scope filtering bound payload; revisit if catalogs grow large.
- **ICU subset scope**: plural/select categories differ per locale —
  validator warns; full ICU deferred.
- **Bundle/request cost** (T-063): per-isolate `Ref`-stamped cache,
  dependency-light renderer, no message data in the bundle; resolver,
  validator, and panel fns stay pure for the 100% coverage gate.

## Links

ADR-0003 (decision record), PL-008 (execution plan). P-065
(locale-native wedge), P-070 (generic kinds), P-081 (portal
instances), P-082 (scoped definitions precedent), T-082 (copy
decisions), P-080 (audit feed for edit history), T-063 (bundle
budget).
