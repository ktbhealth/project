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
  renderer/validator as pure functions (`{param}` interpolation +
  formatter hints via `Intl.*`, `plural`/`select` via
  `Intl.PluralRules`; param signature extraction). Two kinds: `text`
  (plain) and `rich` (sanitized HTML — tag/attr allowlist enforced on
  write AND render). Unit tests to 100% before anything consumes it.
- `I18n` `Context.Tag` + `Layer`: `t(key, params)` (pure over a loaded
  catalog map) and `catalog(scope)` loader; per-isolate cache =
  `Ref` version stamp invalidated on write.
- `plugins/i18n.tables`: `i18n_namespaces` (id, plugin, manifest hash,
  default locale), `i18n_messages` (namespace.key × locale × scope →
  materialized value, kind text|rich, status draft|published, params
  signature), `i18n_locales` (code, endonym, fallback, enabled —
  runtime locale rows, no rebuild), `i18n_crdt_updates` (per-message
  Loro op log).
- `plugins/i18n.services`: baseline sync service (import
  `i18n.messages` contributions + JSON baselines → DB, idempotent
  upsert) + version-stamp bump on every write; CRDT service — apply
  op → append to `i18n_crdt_updates` → re-materialize `value`
  (`@loro-crdt/loro`; **P0 spike: verify WASM works in workerd**).
- `@ktbhealth/ui` `RichTextEditor` (PlateJS) — lands in the ui repo
  first, consumed here via git-tag bump once tagged.
- Register plugin ids `i18n.tables`/`i18n.services`/`i18n.panel`/
  `i18n.chrome` in the registry; arch-policy clean.

## Phase 1 — Codegen + baseline pipeline

- `tools/gen-i18n-manifest.mjs`: walk plugin `i18n.messages`
  contributions + baseline message files + DB-declared namespaces →
  key/param manifest JSON (the shared contract for validator, types,
  panel).
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
  behind a `Ref` version stamp; writes bump the stamp. Stamp re-read at
  most every ~15s per isolate + immediately after local writes — `t()`
  never hits the DB. `t()` = O(1) pure lookup + render.
- **Missing-key policy**: scope → locale → default locale → literal
  `namespace.key` sentinel (never blank, never English-in-disguise);
  every literal fallback counted (metric + access log) so gaps are
  measurable; CI report flags non-default-locale gaps.
- **Chunking**: namespace is the unit everywhere — tables, manifest,
  types, panel tree, payload.
- **Lazy loading**: SSR injects only route-needed namespaces; namespace
  slices ride the TanStack loader response (server fns already run per
  navigation — no extra client roundtrip).
- **Locale switch is a HARD full-reload**: `setLocale` clears isolate
  caches and reloads the page — SSR and client never disagree on
  locale/catalog (hydration safety). Only active locale × active
  namespaces ever ship; no per-locale chunk graph.
- **Namespace resolution is automatic**: route → contributing plugins
  → namespaces, derived from the registry — `pages`/`chrome`/`routes`
  contributions already declare which plugin owns which paths, so the
  host maps a matched route to its plugins' `i18n.messages`
  namespaces with zero manual wiring. Host namespaces (`platform.*`,
  app chrome) always load; namespaces of disabled plugins
  (plugin_state) are skipped. Wildcard `pages` entries resolve via
  the same longest-match rule the dispatcher uses.
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

- Cell editing: live preview via **Streamdown** + custom param-chip
  plugin (`{param}`/`{param, hint}` → styled chips with live sample
  values; click chip to edit sample), pseudo-locale toggle,
  base-vs-effective diff, reset-to-baseline per cell/namespace. `text`
  cells use input/textarea; `rich` cells use the `@ktbhealth/ui`
  PlateJS `RichTextEditor` (HTML in/out, sanitized).
- **Concurrency = Loro CRDT** (`@loro-crdt/loro`): per-message CRDT
  doc; panel edits produce ops appended to `i18n_crdt_updates`, then
  `value` re-materializes on the row — concurrent editors merge, no
  409s. Audit event per materialized write (P-080).
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
  into `i18n_messages` via baseline sync; views switch from
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
  curl HTML contains resolved-locale strings with a scoped message
  applied.

## Phase 6 — Portal locale + polish

- Per-portal-instance default/forced locale (clinic portal pinned
  pt-BR) via `i18n_locales`/portal instance fields.
- Pseudo-locale QA mode; audit feed surfacing copy edits; demo-env
  docs (ties to T-095 runbook).
- Optional later: AI-draft translations via LLM port (P-065/T-095
  `AI_STUB`), in-context editing (Tolgee-style), MF2 evaluation.

## Production hardening (decisions locked in review)

- **Stamp TTL** ~15s/isolate; local writes invalidate immediately —
  `t()` never queries per call.
- **Missing key** → scope → locale → default locale → `namespace.key`
  sentinel (never blank/English-in-disguise); literal fallbacks counted
  (metric + access log); CI gap report for non-default locales.
- **Plural categories**: CLDR plural rules baked into `@ktb/i18n` by
  one-time codegen as a static, versioned table (DECIDED: no
  per-locale override in v1); `Intl.PluralRules` can't enumerate
  categories, so the validator enforces coverage against the table.
- **Rich text IS supported**: `rich` message kind = sanitized HTML via
  the `@ktbhealth/ui` PlateJS `RichTextEditor` (added upstream in the
  ui repo, consumed by git-tag bump); allowlist enforced on write and
  render — never ad-hoc HTML.
- **Concurrency = Loro CRDT** (`@loro-crdt/loro`): per-message doc,
  op log in `i18n_crdt_updates`, materialized `value` on the row for
  fast reads; CRDT only on the edit path. P0 spike verifies WASM in
  workerd; cursors/presence optional later.
- **Param signatures** carry formatter hints (`number`, `date`,
  `datetime`, `relative`, `list`) rendered via `Intl.*`.
- **Locale switch = hard full-reload** (cache clear + reload) —
  hydration never mixes locales.
- **Namespace slices ride loader data** — no separate client fetch
  roundtrip on navigation.

## Delivery workflow (applies to every phase)

- **One worktree + branch + PR per unit of work**: phases 0–3 and 5–6
  are sequential single PRs; phase 4 is one PR per plugin catalog
  migration (each plugin = its own worktree/branch/PR so reviews stay
  small and catalog migrations don't block each other).
- Worktree per unit under the repo's normal flow (`git worktree add`),
  branch named `feat/i18n-<unit>`; never commit ledger changes and
  product changes in the same PR.
- **PR + review gate**: every unit lands via PR with `bun run check`
  green + scoped vitest; reviewer (human or AI-review per PL-004)
  comments → adjust → re-review. **Max 3 review iterations per unit**;
  on the 3rd unresolved round, stop and escalate to a design note in
  the proposal rather than iterating forever.
- **Cleanup**: after merge, remove the worktree (`git worktree
  remove`), delete the local+remote branch, verify no unpushed
  commits / stashes / nested `.git` (repo-deletion checklist applies
  to worktrees too). `myco item set <unit> --status done` + handoff
  entry when the unit's PR merges.
- Phase 4 ordering: `doctors` first (smallest, reference), then by
  catalog size ascending; each migration PR also deletes that
  plugin's `i18n.ts` and updates its `PageContext` consumers.

## Verification (every phase)

- `cd <pkg> && bunx vitest run` scoped per package (global 100%
  threshold — don't chase it); `bun run check` before commit.
- Phase gates: P0 parser/validator 100%; P1 CI drift check fails on
  stale types; P2 SSR smoke per shell (app + platform + portal
  instance); P3 import validation report; P4 zero `i18n.ts` imports
  remain for migrated plugins; P5 zero `paraglide` imports remain.
