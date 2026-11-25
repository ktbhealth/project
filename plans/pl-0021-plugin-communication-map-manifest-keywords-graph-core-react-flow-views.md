+++
id = "PL-021"
title = "Plugin communication map: manifest keywords, graph core, React Flow views"
status = "done"
created = "2026-11-16"
owner = "devin"
priority = "med"
labels = ["plugins"]
links = ["P-082"]
+++

## Execution plan — P-082 plugin communication map

Interactive React Flow map of the plugin registry inside
`/platform/plugins`: manifest `keywords`, pure graph derivation in
`logic.ts`, and three architecture views (Dependencies / Channels /
Domains) with clickable nodes opening the full manifest. Order:
metadata first (independently shippable), then the pure core, then UI.

### Phase 1 — Manifest keywords

- [ ] `@ktb/kernel` `core.ts`: `PluginMeta.keywords?: readonly string[]`
      + doc comment (serializable, parity-channel like the rest of `meta`)
- [ ] `PLUGIN_KEYWORDS` vocabulary export + `keywords.test.ts` over the
      real client registry: every manifest's keywords ⊆ vocabulary and
      non-empty
- [ ] Fill keywords for all ~70 manifests — every
      `plugins/*/src/index.ts` + `src/client.ts` mirror pair and the
      `@ktb/platform` manifests; first keyword = domain
- [ ] `PluginsPage` toolbar search also matches keywords
- Verify: `cd packages/kernel && bunx vitest run` + parity test green;
  `bun run check`

### Phase 2 — Graph derivation core (`plugins/manager/src/logic.ts`)

- [ ] Types: `GraphEdge { source, target, kind, label? }`,
      `PluginNodeData`, `PluginGraph { nodes, edges }`
- [ ] `REGION_OWNERS` prefix table (`portal.*` → `portal.shell`,
      `platform.*`/`console.*` → `platform.panel`, fallback → host
      pseudo-node) + pseudo-node ids
- [ ] `graphFromPlugins(plugins)` — required/optional dep edges, chrome
      `region` edges, generic `contributions[]` kind edges via
      descriptor `consumers`; per-pair aggregation
- [ ] `layeredLayout(graph)` — longest-path depth → columns, registry
      (topo) order within a layer; deterministic
- [ ] `focusNeighborhood(graph, id)` — wraps `requiredClosure` +
      `dependentClosure`
- [ ] `logic.test.ts`: edge kinds, region resolution + unknown
      fallback, layering invariant (depth(dep) < depth(dependent)),
      closures; registry-level region-prefix test
- Verify: `cd plugins/manager && bunx vitest run` (100% gate)

### Phase 3 — Map view + interactive nodes

- [ ] `views/PluginsMapPage.tsx`: design-system `Canvas`
      (`ai-elements/canvas`) + `Controls` + `MiniMap` + legend `Panel`;
      lazy-mounted under a `Tabs` pane on `PluginsPage`
- [ ] Custom node component on `ai-elements` `Node` card: `meta.icon`,
      name, status dot (enabled/disabled/blocked/system), keyword
      `Badge`s; source/target `Handle`s
- [ ] View switcher (`ToggleGroup`): Dependencies / Channels / Domains
      — each remaps edges + node grouping per the proposal spec
- [ ] Manifest `Sheet` on node click: identity / runtime (reuse
      toggle + cascade `AlertDialog` + `ConfigDialog`) / deps as
      focus-buttons / contributions
- [ ] Neighborhood focus: hover/selection dims non-neighbors;
      search + status filter shared with the table toolbar semantics
- [ ] `i18n.ts` catalog: map copy in en / pt-BR / es
- [ ] `PluginsMapPage.test.tsx`: every registry plugin rendered once,
      click opens manifest sheet, status paints
- Verify: `bun run check`; `cd plugins/manager && bunx vitest run`;
  manual `bun run dev` pass over the three views on the real registry
