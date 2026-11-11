+++
id = "P-058"
title = "Dynamic plugin enable/config: /platform/plugins manager + plugin_state"
status = "done"
created = "2026-11-11"
priority = "med"
labels = ["plugins"]
links = ["P-031", "P-050", "P-053"]
+++

## Context

Plugins today are compile-time static: `import.meta.glob` discovers `plugins/*/src/{index,client,api}.ts` into server + client registries, topo-sorted by `requiredDeps`/`optionalDeps`. There is no enable/disable or config concept. workerd cannot do runtime module loading — and that is a feature (type safety, arch lint). So "WordPress-like" means: **plugins ship in the bundle; the platform toggles them on/off and configures them at runtime.** Runtime code upload is out of scope; a future step could shard plugins across isolates, but the registry contract stays.

## Goal

A `/platform/plugins` console page (sidebar item, System group) to manage every discovered plugin: enable/disable with dependency-aware cascade confirmation, per-plugin config, and status (enabled / disabled / blocked-by-dep). Plus a **custom pages plugin** — DB-stored sidebar items + simple pages, the honest "user-authored extension" escape hatch that needs no code.

`plugins/manager` dogfoods the system: it is itself a normal plugin contributing `panel.sidebar` + `panel.pages`.

## State

Host-owned table in `@ktb/db` (not a plugin table — chicken-and-egg):

```sql
plugin_state (plugin_id PK, enabled bool, config jsonb, updated_at)
```

Seeded for every discovered plugin. Config is per-deployment global now; **per-tenant scoping lands with P-053** (tenant column then, not now).

## Enable/disable semantics per contribution kind

| Kind | Mechanism |
|---|---|
| `landingNav`, `landingFooter`, `panel.sidebar`, `panel.headerActions` | Client registry filters by an enabled-ids set fetched per page load — pure render |
| `panel.pages` | Existing `guard` hook → host injects an automatic enabled-check in `platformPageDataFn`; disabled = 404 |
| `landing` (whole-page slot) | Skip disabled plugins when picking first landing plugin (orderPlugins order preserved) |
| `routes` | Can't unmount virtual routes (build-time); gate inside via a `pluginEnabled(pluginId)` helper in loaders → 404 |
| `services` / CQRS | Layers always compose (ManagedRuntime can't recompose per toggle); CQRS dispatch middleware + service entry points fail `PluginDisabled` when disabled |
| `tables` | Never dropped — migrations stay applied; UI shows "data retained" |

Net effect: toggles fully control UI surface + access; service-layer gating is fail-closed dispatch, not uncomposition.

## Dependency handling

- Disabling a plugin that others `requiredDeps`-depend on opens an **alertdialog** listing the cascade set; confirm disables the whole dependent closure (or cancel).
- Enabling a plugin whose required dep is disabled → blocked state in UI with the missing dep named; enabling is refused until the dep is on (no silent auto-enable).
- Dep graph comes from the existing `requiredDeps`/`optionalDeps` declarations — no new metadata needed for this.

## Config

- Optional `meta.config` on the `Plugin` contract: an effect/Schema (or JSON-schema descriptor) + defaults. The Plugins page auto-generates a settings form — write schema once, get UI free.
- Stored as JSON in `plugin_state.config`; read via a `PluginConfig` Effect service provided in core, resolved **per-request** so changes apply without redeploy/isolate restart.
- Plugins with config too complex for generated forms later get a `panel.settings` contribution rendering their own UI (per-plugin settings screen).
- `Plugin` gains optional `meta` (name, description, version, icon) for the management list.

## Server surface

`plugins/manager` fns, all `PlatformAdmin`-gated (same pattern as `platformAdminFn`):

- `listPluginsFn` — registry metadata + contribution kinds + dep edges + current state
- `setPluginEnabledFn` — validates dep closure, applies cascade set atomically
- `getPluginConfigFn` / `setPluginConfigFn` — schema-validated read/write

## Lazy-loading / bundle cost

User flagged: a smart lazy-loader matters as the plugin set grows. `import.meta.glob` supports lazy (`{ eager: false }`) — client-side panel/landing components can code-split per plugin so disabled plugins cost ~nothing in the browser bundle. Server side stays eager (workerd isolate memory is the constraint; revisit if plugin count makes it painful). Phase 2 candidate, not v1.

## Open questions

- Default state for newly discovered plugins: opt-in (safer) vs opt-out (current behavior)? Lean opt-out for v1 to avoid changing existing behavior.
- Does disabling need to gate `landing`/`landingNav` per-user or is it global-only for now? Global-only.

## Non-goals

- Runtime plugin code upload / third-party plugins.
- Per-tenant plugin state (P-053 follow-up).
- Auto-generated config UI for arbitrary nested schemas beyond what effect/Schema can describe — start with flat scalar/enum fields.
