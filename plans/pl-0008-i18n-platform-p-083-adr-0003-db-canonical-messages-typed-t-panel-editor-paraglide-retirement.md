+++
id = "PL-008"
title = "i18n platform (P-083, ADR-0003): DB-canonical messages, typed t(), panel editor, Paraglide retirement"
status = "open"
created = "2026-11-16"
priority = "high"
links = ["T-063", "T-082", "P-065", "P-070", "P-081", "P-082", "P-083", "ADR-0003"]
+++

Execution plan for P-083 / ADR-0003. Conventions: ports/adapters at the
edge, plugin contract single-concern fields + generic `contributions`
(P-070), rbac-scoped panel writes, 100% vitest coverage gate, workerd
bundle budget (T-063). All custom code stays pure/testable.

## Phase 0 — `@ktb/i18n` core + schema

- `packages/i18n` port package: message model + MF1-subset parser/
  renderer/validator as pure functions (`{param}` interpolation,
  `plural`/`select` via `Intl.PluralRules`/`Intl.*`; no markup; param
  signature extraction). Unit tests to 100% before anything consumes it.
- `I18n` `Context.Tag` + `Layer`: `t(key, params)` (pure over a loaded
  catalog map) and `catalog(scope)` loader; per-isolate cache =
  `Ref` version stamp invalidated on write.
- `plugins/i18n.tables`: `i18n_namespaces` (id, plugin, manifest hash,
  default locale), `i18n_messages` (namespace.key × locale × scope →
  value, status draft|published, params signature), `i18n_locales`
  (code, endonym, fallback, enabled — runtime locale rows, no rebuild).
- `plugins/i18n.services`: seed sync service (import `i18n.messages`
  contributions + JSON seeds → DB, idempotent upsert).
- Register plugin ids `i18n.tables`/`i18n.services`/`i18n.panel`/
  `i18n.chrome` in the registry; arch-policy clean.

## Phase 1 — Codegen + seed pipeline

- `tools/gen-i18n-manifest.mjs`: walk plugin `i18n.messages`
  contributions + seed bundle → key/param manifest JSON (the shared
  contract for validator, types, panel).
- `tools/gen-i18n-types.mjs`: manifest → `i18n-messages.d.ts` typed
  `t()` signatures (closed key union + per-key param shapes, craft-ts
  style). CI step: regen + `git diff --exit-code` before typecheck;
  ordering vs `gen:route-types` documented.
- **Defaults are DB rows, not a bundle**: baseline copy lands in
  `i18n_messages` as `global`/`published` rows via migrations + the
  one-time import — the DB is canonical store AND fallback; no message
  JSON ships in the worker bundle.
- One-time importer: `apps/web/messages/*.json` → `i18n_messages`
  (en/es/pt-BR, global scope, published).

## Phase 2 — Resolution + host wiring

- Scoped resolver: portal instance → tenant → global, then locale
  fallback chain from `i18n_locales`; published rows only; draft rows
  visible to editors in panel preview only.
- **Caching**: per-isolate immutable `Map` per (scope-chain × locale)
  behind a `Ref` version stamp; writes bump the stamp. `t()` = O(1)
  pure lookup + render; DB reads only on stamp change.
- **Chunking**: namespace is the unit everywhere — tables, manifest,
  types, panel tree, payload.
- **Lazy loading**: SSR injects only route-needed namespaces; client
  fetches per-route namespace slices via server fn on navigation.
  Locale switch stays full-reload → only active locale × active
  namespaces ship; no per-locale chunk graph.
- Request locale resolution: cookie (`ktb-locale`) → `i18n_locales`
  enabled check → default; replaces Paraglide `requestLocale` internals
  (same external contract during transition).
- Server fns / loaders get catalog via `I18n` service; `PageContext`/
  `PanelPageContext`/`ChromeProps` carry the resolved catalog (or a
  `t` bound to scope) instead of bare `locale` string — additive field,
  migrate consumers incrementally.
- Client: locale switch keeps full-reload semantics; SSR renders the
  request locale — no client i18n provider needed. Hydrated catalog
  payload per route/namespace (payload budget check vs T-063).
- `plugins/i18n.panel` reader page `/platform/i18n`: matrix
  (namespace → key → locale × scope) with badges (draft, missing,
  param-mismatch); rbac read permission.

## Phase 3 — Panel editing

- Cell editing: live preview with sample params, pseudo-locale toggle,
  base-vs-effective diff, reset-to-seed per cell/namespace.
- draft→publish workflow + audit events (P-080 feed integration).
- Bulk ops: copy locale column, fill-missing from fallback.
- Import/export JSON per namespace×locale with validation report
  (reject invalid, apply valid, list errors — nothing partial).
- Write path: server fn validates via effect/Schema + message
  validator; permission `i18n.messages.edit` scoped to data scope
  (global rows = platform-level grant; tenant/portal rows per-scope).
- Missing-translation + param-parity reports.

## Phase 4 — Plugin catalog migration (reference first)

- `I18nCatalog` shim: existing `plugins/*/src/i18n.ts` catalogs import
  into `i18n_messages` via seed sync; views switch from
  `catalog.messages[locale]` to bound `t()` incrementally.
- Migrate smallest catalog first (doctors) end-to-end as reference;
  then patients, consultations, secretaries, etc.
- Chrome label maps (`NAV_LABELS`/`PANEL_LABELS`/`FOOTER_NOTES` in
  `lib/i18n.ts`) → message keys resolved through `t()`.
- Retire each migrated `i18n.ts`; `PATIENT_LOCALES`-style locale lists
  → `i18n_locales` rows.

## Phase 5 — Host Paraglide retirement

- All `messages/*.json` keys live in DB; `m.*` call sites → `t()`
  (codemod for mechanical cases; `lib/i18n.ts` wrappers first).
- Remove `paraglideVitePlugin` from `apps/web/vite.config.ts`, delete
  `src/paraglide/` output + `project.inlang`, drop `@inlang/paraglide-js`
  dep; locale picker reads `i18n_locales`.
- `bun run gen:route-types` + full `bun run check` green; SSR smoke:
  curl HTML contains resolved-locale strings with DB override applied.

## Phase 6 — Portal locale + polish

- Per-portal-instance default/forced locale (clinic portal pinned
  pt-BR) via `i18n_locales`/portal instance fields.
- Pseudo-locale QA mode; audit feed surfacing copy edits; demo-env
  docs (ties to T-095 runbook).
- Optional later: AI-draft translations via LLM port (P-065/T-095
  `AI_STUB`), in-context editing (Tolgee-style), MF2 evaluation.

## Verification (every phase)

- `cd <pkg> && bunx vitest run` scoped per package (global 100%
  threshold — don't chase it); `bun run check` before commit.
- Phase gates: P0 parser/validator 100%; P1 CI drift check fails on
  stale types; P2 SSR smoke per shell (app + platform + portal
  instance); P3 import validation report; P4 zero `i18n.ts` imports
  remain for migrated plugins; P5 zero `paraglide` imports remain.
