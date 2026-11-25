+++
id = "P-109"
title = "Site builder: Puck editor for pages and layouts with builderJson + Plate rich-text field"
status = "done"
created = "2026-11-20"
priority = "med"
links = ["P-094", "P-102", "P-105", "P-106", "P-107", "P-108", "P-111"]
epic = "site-builder"
+++

## Vision

Elementor-like visual editing for the Site group, built on Puck + Plate:

- **Pages** (`custom_pages` → renamed `pages`): add `builderJson` column holding the Puck document; "edit" opens the Puck canvas. Existing `richTextId`/`layoutId`/`searchSchemaId` path keeps working — additive, no migration of content.
- **Layouts**: same Puck editor; layout body becomes a Puck doc (ordered sections/components) referenced by pages via `layoutId`. Edit-once-update-all-pages preserved.
- **Components** (P-108): the block palette inside the editor.
- **Templates** (P-107): "save as template" / "new from template" on both surfaces.
- **Routes** (P-106): page paths register as typed route keys.

## Key correction vs earlier framing

Layouts do NOT overlap/conflict with Puck — Puck is the shared *editor* for both document kinds (page body and layout body are both ordered component trees). The old section-slot layout model can be migrated into builderJson or kept for simple pages; decide in the plan.

## Shape (proposal)

- New plugin `site-builder` (or extend `pages`/`layouts`): Puck `<Puck>` editor route under `/platform`, render via `<Render>` in `CustomPageView`/layout resolver.
- `builderJson` schema-versioned column; `data` migration hooks (Puck supports migrations) for breaking prop changes.
- Rich-text component field embeds Plate (`RichTextEditor` from `@ktbhealth/ui`).
- Rename `custom_pages` → `pages` table + `CustomPages` → `Pages` service (mechanical; do early, before builderJson lands, to avoid renaming twice).

## Dependencies

P-105 (spike validates Puck+tokens+Plate embedding), P-108 (component registry feeds the palette), P-107 (templates), P-106 (route keys). Plan sequence in linked PL.
