+++
id = "P-113"
title = "Migrate @ktbhealth/ui into a product plugin (dual-entry package: ./plugin manifest + ./ lib)"
status = "open"
created = "2026-11-20"
priority = "low"
labels = ["future"]
links = ["P-108", "P-109"]
+++

## Vision (future)

Promote `@ktbhealth/ui` from an external git-tag dependency into a first-class plugin citizen of the product monorepo — while keeping it consumable as a plain library.

## Mechanism

Dual-entry package via `package.json` `exports`:

- `"@ktbhealth/ui"` (or scoped pkg name) → pure component library, no plugin manifest, no registry coupling — usable by any consumer (docs site, website-draft, storybook).
- `"@ktbhealth/ui/plugin"` → subpath exporting the plugin manifest (`elementKinds` contributions for ui primitives, chrome, etc.) — imported only by the product composition root.

This splits bundles: apps that need just components never pull the plugin manifest/registry code; the product gets ui primitives as element kinds for the builder palette (P-108) without vendoring.

## Why future / open questions

- ui is currently an org-level repo consumed via git tags by multiple consumers (product, docs, website) — folding it into product's plugin graph changes release/versioning (ui tags are the org's component contract).
- Alternative: keep ui repo, add a thin `plugins/ui-kit` wrapper in product that contributes element kinds referencing `@ktbhealth/ui` components — likely cheaper than a repo merge. Evaluate both.
- Element kinds for primitives (button, card, input as builder blocks?) vs composite blocks only — decide with P-108.

Status: proposed/future — do not schedule until P-108 proves the registry shape.
