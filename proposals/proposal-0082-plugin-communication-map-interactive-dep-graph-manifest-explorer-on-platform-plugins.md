+++
id = "P-082"
title = "Plugin communication map: interactive dep-graph + manifest explorer on /platform/plugins"
status = "open"
created = "2026-11-16"
owner = "devin"
priority = "med"
labels = ["plugins"]
links = ["P-050", "P-058", "P-062", "P-070", "PL-008"]
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
- Controlled flat vocabulary (no `domain:` prefixes — prefix namespacing
  re-invents fields; the facet comes from position in the list).
  Convention: **first keyword is the domain** (drives Domains lanes),
  the rest are free descriptors. Kernel exports
  `PLUGIN_KEYWORDS` — the documented valid set — and a vitest asserts
  every manifest's keywords ⊆ the set and non-empty, so the taxonomy
  can't rot.

  | facet    | keywords (initial set)                                              |
  | -------- | ------------------------------------------------------------------ |
  | domain   | `patients` `consultations` `transcription` `eval` `portal` `platform` `groups` `notifications` `tenant` `audit` `site` `pages` `dashboard` |
  | role     | `services` `ui` `routes` `shell` `admin` `marker` `tables`         |
  | audience | `doctor` `patient` `secretary` `admin`                             |

- Fill keywords for all ~70 manifests in both `index.ts` and
  `client.ts` mirrors.
- Keywords feed search (map + existing table toolbar), the
  domain-cluster view lanes, and node badges.

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

The map lands as a `Tabs` pane beside the existing table on
`/platform/plugins` (one sidebar item; the same `plugins` + `states`
props feed both panes — no loader changes). Inside the map, a
`ToggleGroup` switches sub-views — each answers one architectural
question:

- **Dependencies** — "what breaks if I turn this off?" Layered topo
  columns; required solid / optional dashed; node color = runtime
  status (enabled / disabled / blocked-by-dep), `system` badge;
  blocked edges (dep → disabled requiredDep) tinted destructive.
- **Channels** — "who talks to whom, through what?" Only `region`
  and `kind` edges drawn; `required`/`optional` fade to ghost so
  structural deps stop dominating. Pseudo-nodes for host surfaces
  show the boundary between plugin space and host space.
- **Domains** — "where are the boundaries?" Nodes grouped into
  swimlanes by first keyword; only cross-domain edges drawn,
  intra-domain edges hidden — coupling between domains is the signal.

Every view shares the toolbar: text search (matches id / name /
keywords), status filter (same semantics as the table), a legend
(`Panel` explaining edge styles + status colors), and a
highlight-on-hover neighborhood. Fit-view / zoom controls via
`Controls`; `MiniMap` from `@xyflow/react` styled on tokens.

### Interactive nodes

Custom node type on the design-system `Node` card (`ai-elements`):
`meta.icon` + display name + status dot + keyword `Badge`s, sized
compact with `NodeHeader`/`NodeContent`. Clicking a node opens a
right-side `Sheet` — the **manifest panel**:

- **Identity** — icon, name, id, description, version, keywords,
  system marker.
- **Runtime** — enabled/disabled/blocked with the same
  enable/disable flow the table row runs (cascade `AlertDialog`
  reuse), `Configure` reusing `ConfigDialog` when `meta.config`
  exists.
- **Dependencies** — required/optional dep ids as buttons that
  focus the target node (fit-view + select), plus the derived
  closures (`requiredClosure`, `dependentClosure`) already computed
  by `logic.ts`.
- **Contributions** — `contributionKinds()` list plus chrome regions
  and generic `contributions[]` entries with their target surface.

Hover/selection dims everything outside the transitive neighborhood
(`focusNeighborhood`) — one click answers "what does this need, and
what needs it".

## i18n

Manager catalog gains the map copy (view names, legend, empty states,
manifest-sheet labels) in all three locales (en, pt-BR, es) — same
`managerMessages(locale)` convention.

## Bundle cost

`@xyflow/react` is already shipped through `@ktbhealth/ui`, but the
map tab should not tax the table view: the canvas mounts lazily
(`React.lazy` + `Suspense`, or mounted only when the Map tab is
active) so the table stays the fast default path. Node/edge objects
derive in `useMemo` from `plugins` + `states` — re-derived only when
the loader data changes (a toggle invalidates the loader → the map
repaints with fresh status, same as the table).

## Rollout

1. **Manifests** — kernel `PluginMeta.keywords` + `PLUGIN_KEYWORDS`
   vocabulary + validation test; fill all ~70 manifests in `index.ts`
   / `client.ts` pairs. Shippable alone: keywords already improve the
   table's search.
2. **Graph core** — `logic.ts` derives `graphFromPlugins`,
   `layeredLayout`, `focusNeighborhood`, `REGION_OWNERS`; full unit
   coverage (100% gate).
3. **Map view** — `PluginsMapPage` (canvas + node type + manifest
   `Sheet` + view switcher + legend), i18n, tabs wiring on
   `PluginsPage`, view test. Reuses existing toggle/config dialogs.

## Testing

- `logic.test.ts`: edge derivation per kind, region-owner resolution
  incl. unknown-region fallback, layering invariants (depth respects
  every required edge), neighborhood focus = the two existing
  closures.
- Keyword vocabulary test over the real registry (imports the client
  registry, not fixtures — rot is caught where it happens).
- View test: rendered map shows every registry plugin exactly once;
  click opens the manifest sheet; disabled/blocked states paint.
- Parity test (`apps/web plugins.test.ts`) unchanged — keywords are
  `meta` data and mirrors must match like the rest of `meta`.
- `bun run check` + scoped `bunx vitest run` in `plugins/manager`.

## Risks

- **Region-owner drift** — new shells/regions appear without updating
  `REGION_OWNERS`. Mitigation: a vitest asserting every
  `chrome[].region` prefix in the registry resolves to a known owner
  (same "test the real registry" pattern as keywords).
- **Readability at ~70 nodes** — layered columns can get tall.
  Mitigation: domain lanes collapse intra-domain edges; search +
  focus mode shrink the visible set; a future barycenter pass and
  per-domain collapse are layout-internal upgrades.
- **Keyword vocabulary drift** — mitigated by the vocabulary test;
  adding a keyword is a one-line kernel change, reviewable.

## Open questions

- Layout: hand-rolled longest-path layering (zero deps, deterministic)
  vs `dagre`/`elkjs` (better edge routing, new dep). Leaning to
  hand-rolled v1 given ~70 nodes and mostly-sparse edges.
- ~~Map placement~~ — decided: `Tabs` pane on `/platform/plugins`.
  If deep-linking a view becomes valuable, search-param state
  (`?view=domains`) is a follow-up that doesn't change the contract.
- Runtime traffic (CQRS/events) is not observable from manifests —
  out of scope, but the edge model leaves room for it later.

## Non-goals

- Editing the graph (drag-to-redeclare deps) — the map is read-only.
- Persisted/manual node arrangement, saved layouts, or export (PNG /
  SVG) — revisit if the map becomes a docs artifact.
- Runtime/event telemetry on edges (CQRS traffic, dispatch counts).
- Per-tenant state visualization (P-053 follow-up).
- A graph DSL or new manifest fields beyond `meta.keywords` — the map
  is a projection of what plugins already declare.
