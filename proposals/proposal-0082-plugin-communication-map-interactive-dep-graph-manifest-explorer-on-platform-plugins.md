+++
id = "P-082"
title = "Plugin communication map: interactive dep-graph + manifest explorer on /platform/plugins"
status = "open"
created = "2026-11-16"
owner = "devin"
priority = "med"
labels = ["plugins"]
links = ["P-050", "P-058", "P-062", "P-070"]
+++

## Context

The plugin registry is now large enough that the `/platform/plugins`
table (P-058) no longer communicates architecture: ~35 plugin packages
contributing ~70 `Plugin` objects, ordered by `requiredDeps` /
`optionalDeps` into the dep graph `orderPlugins`
(`@ktb/driver-framework`) topo-sorts at boot. How plugins actually
talk to each other — service layers behind dep edges, chrome regions
they publish into, generic `contributions[]` kinds other surfaces
collect — lives only in code. The manager UI has dep-closure logic
(`plugins/manager/src/logic.ts`) but renders it as a flat table.

`PluginMeta` (kernel `core.ts`) carries `name`/`description`/
`version`/`icon`/`config` but no domain keywords, so there is no way to
group or filter plugins by architectural concern.

The design system already wraps React Flow: `ai-elements/canvas`,
`node`, `edge`, `controls`, `toolbar`, `panel` (`@xyflow/react` is a
published dep of `@ktbhealth/ui`). No new rendering dependency is
needed.

## Goal

An interactive architecture map inside the manager plugin: a React
Flow canvas where every `Plugin` object is a node and its declared
relationships are edges — navigable (pan/zoom/minimap), clickable
(nodes open the full manifest), and rendered entirely in design-system
components/tokens. Plus `meta.keywords` on every plugin manifest as
the missing grouping dimension, and a small set of purpose-built
views that explain the system's architecture at a glance.

## Design

### Manifest keywords

- `PluginMeta` gains `keywords?: readonly string[]` — plain strings,
  serializable, so client mirrors (`src/client.ts`) declare them
  identically and the server/client parity test keeps holding.
- Fill keywords for all ~70 manifests: a controlled vocabulary —
  domain (`patients`, `consultations`, `eval`, `transcription`,
  `platform`, `portal`, `billing`-adjacent…), role (`services`,
  `ui`, `routes`, `shell`, `admin`, `adapter`, `marker`), and
  audience where relevant (`doctor`, `patient`, `secretary`).
- Keywords feed search, the domain-cluster view, and node badges.

### Graph model (pure derivation, `logic.ts`)

All structure comes from the client registry `ctx.plugins` — already
injected into the panel view — and runtime state from the existing
loader (`PluginStates.entries`). No new server fn for structure.

Edge types derived per plugin pair:

- `required` — `requiredDeps` entries; solid edge, dep → dependent
  (same orientation as `orderPlugins`).
- `optional` — `optionalDeps` entries; dashed edge.
- `region` — `chrome[]` entries publishing into a named region owned
  by a shell plugin (e.g. `portal.sidebar`, `portal.header`); thin
  edge to the region owner.
- `kind` — generic `contributions[]` entries collected by another
  surface (e.g. `eval.*` kinds); dotted edge to the collecting plugin
  where the kind descriptor names one.

`graphFromPlugins(plugins) → { nodes, edges }` plus
`layeredLayout(graph)` — longest-path layering over the topo order,
deterministic, no layout dependency — are pure functions unit-tested
alongside the existing closure walkers.

### Views

A view switcher (design-system `Tabs`/`ToggleGroup`) on
`/platform/plugins` adds a "Map" surface beside the table, with
sub-views:

- **Dependencies** — layered columns by topo depth; required solid /
  optional dashed; node color = runtime status (enabled / disabled /
  blocked-by-dep), `system` badge.
- **Communication** — region + kind edges emphasized: who publishes
  into which chrome region, which services plugins the dependents
  consume, generic contribution flow.
- **Domains** — nodes clustered into keyword lanes (first domain
  keyword = lane); only cross-domain edges drawn, making domain
  boundaries and coupling visible.

### Interactive nodes

Custom node type on the design-system `Node` card: `meta.icon` +
display name + status indicator + keyword badges. Clicking a node
opens a `Sheet`/`Dialog` with the full manifest — meta (incl.
keywords), `config` schema, contribution kinds, declared deps with
jump-to-node links — and reuses the existing enable/disable and
config dialogs so the map is a management surface, not a picture.
Neighbor highlighting on hover/focus (transitive deps + dependents,
which `logic.ts` already computes).

## i18n

Manager catalog gains the map copy (view names, legend, empty states,
manifest-sheet labels) in all three locales (en, pt-BR, es) — same
`managerMessages(locale)` convention.

## Open questions

- Layout: hand-rolled longest-path layering (zero deps, deterministic)
  vs `dagre`/`elkjs` (better edge routing, new dep). Leaning to
  hand-rolled v1 given ~70 nodes and mostly-sparse edges.
- Map as a `Tabs` pane inside `/platform/plugins` vs a second
  `panel.pages` entry (`/platform/plugins/map`) — tabs keep one
  sidebar item; a page enables deep-linking.
- Runtime traffic (CQRS/events) is not observable from manifests —
  out of scope, but the edge model leaves room for it later.

## Non-goals

- Editing the graph (drag-to-redeclare deps) — the map is read-only.
- Runtime/event telemetry on edges.
- Per-tenant state visualization (P-053 follow-up).
