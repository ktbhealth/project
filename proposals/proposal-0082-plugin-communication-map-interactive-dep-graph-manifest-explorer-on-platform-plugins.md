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

**Nodes** — one per `Plugin` object plus *host pseudo-nodes* for
surfaces that consume contributions but are not plugins (app
marketing shell, `apps/eval` CLI). Pseudo-nodes render dashed/muted
and are sinks only.

```ts
interface PluginNodeData {
  plugin: Plugin;              // manifest source of truth
  status: "enabled" | "disabled" | "blocked" | "system";
  missing: readonly string[];  // disabled requiredDeps, for "blocked"
  kinds: readonly string[];    // contributionKinds() — already exists
  depth: number;               // longest-path layer, for layout
}
```

**Edges** — derived per plugin pair, aggregated: at most one edge per
(source, target, kind); a pair carrying several kinds gets parallel
edges or one composite edge whose style follows the strongest kind
(required > region > kind > optional).

| kind       | source                                                        | style  |
| ---------- | ------------------------------------------------------------- | ------ |
| `required` | `requiredDeps` — dep → dependent (same orientation as `orderPlugins`) | solid  |
| `optional` | `optionalDeps` — ordering hint only                            | dashed |
| `region`   | `chrome[]` entry's `region` field → region owner               | thin   |
| `kind`     | `contributions[]` entry → collecting surface named by the kind descriptor's `consumers` | dotted |

**Region ownership** — chrome `region` ids are shell-owned slot
names, so `logic.ts` carries a static `REGION_OWNERS` table mapping
prefixes to owners: `portal.*` → `portal.shell`, `platform.*` /
`console.*` → `platform.panel`, `app.*` / unknown → the host
pseudo-node. Unknown region ids degrade to the host node rather than
being dropped — the map never hides declared communication.

**Kind edges** — `ContributionKindDescriptor.consumers` already names
the collecting surface per kind (`app-host.server`, `plugin-manager`,
`eval-cli`, …); consumers that map to a plugin id draw a real edge,
host consumers draw to the pseudo-node.

**Layout** — `layeredLayout(graph)`: longest-path depth per node
(sinks at the far right), x = depth × column width, y = order within
layer following registry (topo) order so adjacent dependents cluster.
Deterministic, dependency-free, and unit-tested next to the existing
closure walkers. One optional barycenter pass can reduce crossings
later without changing the contract.

```ts
graphFromPlugins(plugins) → { nodes: Map<PluginId, PluginNode>, edges: GraphEdge[] }
layeredLayout(graph)        → Map<PluginId, { x, y }>
focusNeighborhood(graph, id) → { upstream: Set, downstream: Set }   // reuses requiredClosure/dependentClosure
```

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
