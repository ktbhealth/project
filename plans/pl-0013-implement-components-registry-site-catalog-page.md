+++
id = "PL-013"
title = "Implement components registry + Site catalog page"
status = "done"
created = "2026-11-20"
owner = "devin"
priority = "med"
links = ["P-108", "P-109", "PL-014"]
epic = "site-builder"
+++

# Plan: components registry (P-108)

## Phase 1 — Registry contract (decided)
- Reuse the existing `elementKinds` contribution (P-102) — extend `ElementKind` in `@ktb/kernel` with optional `fields` (prop schema → palette docs + validation). NO new `builder.components` kind; Puck is one consumer, the registry stays editor-agnostic.
- New plugin `plugins/builder-blocks` ships the core primitives as element-kind contributions (`src/index.ts` manifest + `src/client.ts` renderer/editor, rich-text pattern).
- `plugins/site-builder` (PL-014) compiles the ElementKinds registry → Puck config; blocks never import Puck types directly.

## Phase 2 — Core block set (`plugins/builder-blocks`)
- Layout primitives: section, container, columns, divider/spacer.
- Content: heading, CTA/button.
- Data: metric/stat, simple table.
- NOT included: `rich-text` (owned by `plugins/rich-text`, already an element kind) and `image` (owned by `plugins/image`, P-111/PL-016) — the palette composes all kinds from the registry automatically.
- All render through `@ktbhealth/ui` + Tailwind tokens — no hardcoded styles.

## Phase 3 — Site catalog page
- `/platform/components` sidebar item (Site group): browsable catalog — name, description, category, live preview, prop schema.
- Read-only v1; "insert into page" wiring lands with P-109 editor.

## Phase 4 — Plugin-contributed blocks (stretch)
- One domain example: e.g. patients `PatientCard` or consultations summary block — proves plugins can contribute element kinds with builder fields.
- Placement rule: composite/domain blocks live in their product plugin; pure ui primitives would come from `@ktbhealth/ui` (P-113, future) — not vendored here.

## Phase 5 — Verify
- Every block renders in catalog preview AND inside Puck canvas (once P-109 editor exists) with identical output.
- Token check: no hex literals; dark/light both correct.
