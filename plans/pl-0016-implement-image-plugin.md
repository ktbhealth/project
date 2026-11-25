+++
id = "PL-016"
title = "Implement image plugin"
status = "done"
created = "2026-11-20"
owner = "devin"
priority = "med"
links = ["P-108", "P-111", "PL-014"]
epic = "site-builder"
+++

# Plan: image plugin (P-111)

## Phase 1 — Storage + table
- `images` table `{ id, title, alt, mime, size, storageKey, width, height, createdAt }` (drizzle, plugin tables contribution).
- Storage port + adapters: R2 prod / local fs-or-stub dev; serve route `/api/images/:id` with cache headers. Evaluate reuse of theme Phase-4 asset pipeline first.

## Phase 2 — elementKinds contribution
- `kind: "image"`: renderer = responsive image via `@ktbhealth/ui` tokens; editor = library-picker dialog (grid + upload + alt/fit fields).
- Register server descriptor in `ElementKinds` service (mirror rich-text's `*ElementKindLive` pattern).

## Phase 3 — Admin library page
- `/platform/images` (Site group sidebar item): grid, upload, edit meta, delete.
- Delete-hook contributions to `*-delete.actions` regions so pages/layouts/rich-text warn on referenced images (mirror pages/layouts cleanup actions).

## Phase 4 — Builder integration
- Verify `image` kind appears in P-108 palette automatically; optional thin builder-block wrapper if fields mapping needs it.
- Rich-text inline images: evaluate embedding picker in Plate (phase 2 if non-trivial).

## Phase 5 — Verify
- Upload → serve → render round-trip; alt text required-ness decision; delete-with-references flow.
