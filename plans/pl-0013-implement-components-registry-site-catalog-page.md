+++
id = "PL-013"
title = "Implement components registry + Site catalog page"
status = "open"
created = "2026-11-20"
priority = "med"
links = ["P-108", "P-109"]
epic = "site-builder"
+++

# Plan: components registry (P-108)

## Phase 1 — Registry contract
- Decide: own registry compiling to Puck config (preferred — editor-agnostic) vs direct Puck config. Own registry: `{ id, label, description, category, fields(schema), render, preview? }`.
- Contribution point in kernel (`builder.components` region or plugin manifest field); declaration-merged typing for component ids.

## Phase 2 — Core block set
- Layout primitives: section, container, columns.
- Content: heading, rich-text (embeds Plate `RichTextEditor`), image, CTA/button.
- Data: metric/stat, simple table.
- All render through `@ktbhealth/ui` + Tailwind tokens — no hardcoded styles.

## Phase 3 — Site catalog page
- `/platform/components` sidebar item (Site group): browsable catalog — name, description, category, live preview, prop schema.
- Read-only v1; "insert into page" wiring lands with P-109 editor.

## Phase 4 — Plugin-contributed blocks (stretch)
- One domain example: e.g. patients `PatientCard` or consultations summary block — proves plugins can contribute.
- Split decision: primitives live in `@ktbhealth/ui`, domain blocks in plugins.

## Phase 5 — Verify
- Every block renders in catalog preview AND inside Puck canvas (once P-109 editor exists) with identical output.
- Token check: no hex literals; dark/light both correct.
