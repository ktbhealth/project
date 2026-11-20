+++
id = "PL-014"
title = "Implement site builder: Puck editor for pages and layouts"
status = "open"
created = "2026-11-20"
priority = "med"
blocks = ["T-190", "PL-017"]
links = ["T-190", "P-102", "P-105", "P-106", "P-107", "P-108", "P-109"]
epic = "site-builder"
+++

# Plan: site builder (P-109) — sequenced

## Phase 0 — Spike (P-105)
- Embed `<Puck>` in scratch route; register 3-4 `@ktbhealth/ui` components + Plate-backed rich-text field.
- Validate: token fidelity (light/dark), JSON schema, bundle cost, DX. Go/no-go.

## Phase 1 — Rename (T-190)
- `custom_pages` → `pages` table, `CustomPages` → `Pages` service, `CustomPageView` → `PageView`, route/plugin ids where user-facing. Migration + all references. Do BEFORE builderJson to avoid double rename.

## Phase 2 — Pages builderJson
- Add `builderJson` (versioned) column to `pages`; `PagesAdminConsolePage` gains "Open in builder" → Puck editor route `/platform/pages/:id/edit` (or side panel).
- `CustomPageView` render path: builderJson → `<Render>`; fallback to richTextId/layoutId path.
- Puck data migrations registered for schema versioning.

## Phase 3 — Layouts builderJson
- Same editor for `layouts` rows; page `layoutId` resolution renders layout's Puck doc.
- Decide fate of old section-slot model: migrate to builderJson or keep dual-path (recommend migrate, keep reader for legacy rows).

## Phase 4 — Integrations
- Components palette from P-108 registry.
- "Save as template"/"new from template" via P-107 (`page`/`layout` kinds).
- Page paths register route keys via P-106.

## Phase 5 — Verify
- E2E: create page in builder → publish → renders at path; edit layout → all bound pages update.
- Archive/rollback: builderJson absent → old render path intact.
