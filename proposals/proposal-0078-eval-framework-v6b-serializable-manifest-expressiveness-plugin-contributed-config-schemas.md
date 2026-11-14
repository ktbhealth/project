+++
id = "P-078"
title = "Eval framework v6b: serializable manifest expressiveness — plugin-contributed config schemas"
status = "open"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["P-074", "P-075", "P-076", "P-077"]
+++

Manifest expressiveness ceiling: today manifests are pure data but shallow — stages/metrics/gates are id refs with flat params. Push expressiveness WITHOUT losing diff-ability: **plugins contribute serializable config schemas**, manifests instantiate them.

## Problem

A suite author can pick stages/metrics/gates but can't express *how* they're configured beyond flat JSON params — no nested structure, no per-metric options, no reusable gate presets, no validation beyond eval-core's generic Schema.

## Proposal: contribution-declared config schemas

- Each contribution (stage/metric/gate/capability) declares a **serializable config Schema** alongside its implementation — e.g. `wer` metric contributes `{schema: {normalize: bool, caseFold: bool}, impl: fn}`.
- Manifest entries become `{id, config: <schema-validated data>}` instead of bare ids + flat params — config is still pure JSON data, validated against the contribution's own Schema at `eval list`/`run` time.
- Gate presets: domain packs contribute named gate bundles (`consult.strict-gates` = [wer-floor, entity-precision, negation, harmful-edits]) — manifests reference the preset or override individual params. Composability via named, versioned presets.
- Condition contributions (for P-077 DAG): `{id: "wer-above", schema: {threshold, stage}}` — serializable, plugin-implemented.

## Rules preserved

- Manifests remain pure data — no functions, no imports beyond defineSuite.
- Config schemas live with the contribution (domain pack owns its gate/metric config shapes) — eval-core stays domain-blind.
- Unknown config keys rejected (Schema exactness) — manifests can't drift silently.

## Example delta

```ts
// before
gates: [{id: "auto-accept", params: {target: 0.9, minN: 40}}]
// after
gates: [{id: "auto-accept", config: {target: 0.9, minN: 40,
        bands: [{lo: 0.9, hi: 1.0}], policy: "consult.strict"}}]
metrics: [{id: "wer", config: {normalize: true}}]
stages: [{id: "judge", config: {questions: "consult.entity-preservation"}}]
```

## Acceptance

- Contributions declare config Schemas; manifests validate against them at load.
- Gate presets compose; individual params still overridable.
- eval-core gains zero domain knowledge — all config shapes come from contributors.
