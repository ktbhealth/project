+++
id = "ADR-0003"
title = "UI copy as runtime data: DB-canonical i18n via custom @ktb/i18n port"
status = "open"
created = "2026-11-16"
priority = "high"
links = ["T-082", "P-065", "P-070", "P-081", "P-082", "P-083", "PL-008"]
+++

# ADR-0003: UI copy as runtime data — DB-canonical i18n

## Decision

All UI copy is **data**, not code. Messages live in `i18n_messages`
(DB: `namespace.key × locale × scope → value`, `draft|published`),
scoped `global | tenant | portal instance`. A custom `@ktb/i18n`
Effect port resolves `t(key, params)` per request; codegen emits typed
key/param signatures and the key manifest; baseline copy is DB-seeded
(`global`/`published` rows via migrations + import) so the DB is both
canonical store and fallback — no message JSON ships in the worker
bundle. Paraglide and the per-plugin
`i18n.ts` catalogs are retired after baseline import. Copy is edited in a
`/platform/i18n` matrix panel under rbac-scoped permissions.

## Rationale

- **Product fit**: multi-tenant, white-label (P-082), locale-native
  wedge (P-065), per-portal wording (P-081) all want runtime-editable,
  per-scope copy. New locales must not require a rebuild.
- **Library survey** (full record in P-083): compile-time libs
  (Paraglide, Lingui, typesafe-i18n) share the model mismatch; i18next
  and FormatJS are heavy and unscoped; Tolgee is a whole
  Spring/Postgres service; no Effect-native option exists;
  `Intl.MessageFormat` is TC39 Stage 1. The novel surface — scoped
  resolution, DB store, codegen, validation — has no off-the-shelf
  answer; commodity parts stay native (`Intl.PluralRules`, `Intl.*`,
  ~zero-cost React binding since locale switch reloads).
- **Safety is relocated, not lost**: typed `t()` + param signatures
  via `gen:i18n-types`, CI regen+diff drift check, write-path
  validation (MF1-subset parser doubles as validator), draft→publish
  + audit (P-080) replace PR review for copy.

## Consequences

- Copy review moves from PRs to draft→publish workflow — needs
  discipline; optional scheduled git export keeps a diffable history.
- Type safety becomes CI-time: `gen:i18n-types` must run before
  typecheck; ordering with `gen:route-types` must be wired.
- Baseline copy is DB-seeded (`global`/`published` rows via migrations
  + import): the DB is canonical store and fallback — no message JSON
  in the worker bundle. Namespace chunking + per-route lazy slices +
  per-isolate `Ref`-stamped cache bound payload and request cost.
- Message format is an in-house ICU MF1 subset (`{param}`,
  `plural`/`select` via `Intl.PluralRules`); full ICU via formatjs
  parser remains an upgrade path. MF2 deferred.
- `m.*` → `t()` migration across routes is the long pole (P-083 P3);
  plugin catalogs migrate first via an `I18nCatalog` shim.
- Workerd budget (T-063): per-isolate `Ref`-stamped cache (stamp
  re-read ≤ ~15s/isolate), dependency-light renderer; resolver/
  validator/panel stay pure for the 100% coverage gate.
- Hard requirements: locale switch is a full reload (hydration
  safety); missing keys fall back scope → locale → default → key
  literal; writes merge via Loro CRDT (`@loro-crdt/loro`, op log in
  `i18n_crdt_updates`, materialized value on the row) — no optimistic
  locks; rich messages are sanitized HTML edited via the
  `@ktbhealth/ui` PlateJS `RichTextEditor`; plural-category coverage
  uses a CLDR-derived table (`Intl.PluralRules` can't enumerate
  categories); namespace slices ride loader data — no separate client
  fetch.
