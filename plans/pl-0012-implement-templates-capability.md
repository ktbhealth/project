+++
id = "PL-012"
title = "Implement templates capability"
status = "done"
created = "2026-11-20"
owner = "devin"
priority = "med"
links = ["P-107", "PL-014"]
epic = "site-builder"
done = "2026-11-24"
+++

# Plan: templates capability (P-107)

## Phase 1 — Registry infra
- New `packages/templates` (or `plugins/templates` if it needs registry contributions): `templates` table `{ id, kind, name, description, payload(json), createdBy, timestamps }`.
- `Templates` service: admin-gated `adminList/listByKind/getById/save/instantiate/delete`; kind registry `{ kind, serialize, deserialize, validate, preview? }`.
- Follow `@ktb/schemas`/`@ktb/layouts` patterns: serviceOption degradation, delete-hook cleanup contributions.

## Phase 2 — Admin surface
- `/platform/templates` console page (Site or new group): list grouped by kind, create/rename/delete.
- "Save as template" dialog-action region (mirror `rich-text.delete.actions` pattern) so any surface can offer it.

## Phase 3 — First kinds
- `layout` kind (from `plugins/layouts`) — lowest risk, proves the pattern.
- `page` kind once P-109 lands builderJson (payload = Puck doc + page defaults minus path/published).
- Later kinds backlog: rich-text doc, form, eval dataset.

## Phase 4 — Verify
- Round-trip: save layout → instantiate → identical render.
- Cross-plugin: second kind proves genericity.
