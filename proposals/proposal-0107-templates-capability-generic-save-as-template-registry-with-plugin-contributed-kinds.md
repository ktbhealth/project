+++
id = "P-107"
title = "Templates capability: generic save-as-template registry with plugin-contributed kinds"
status = "open"
created = "2026-11-20"
priority = "med"
links = ["P-105"]
+++

## Vision

Templates as a **platform capability**, not a page feature: a `templates` registry any plugin can plug into. Each plugin declares a *kind* (shape + serializer + validator); any admin surface can "save as template" and "insert from template" for that kind. Pages and layouts are the first two kinds; future kinds: consult notes, emails, eval fixtures, wizard flows, etc.

## Why

- Reuse is cross-domain: the same mechanism that templates a landing page can template a clinical note or an eval dataset.
- Plugin architecture already has this shape: `schemas`, `layouts`, `rich-text` registries with admin CRUD + delete-hook cleanup — templates follow the same pattern.
- Keeps Puck/page concerns out of the generic mechanism: templates store opaque kind-specific payloads.

## Shape (proposal)

- `templates` plugin (infra, like `packages/schemas`): table `templates { id, kind, name, description, payload(json), createdBy, timestamps }` + `Templates` service (admin-gated CRUD + `listByKind`/`instantiate`).
- Kind contract: plugins register `{ kind, serialize, deserialize, validate, renderPreview? }` via a registry contribution.
- First kinds: `page` (Puck builderJson), `layout` (Puck builderJson), later `rich-text`, `form`, `eval-dataset`.
- Admin UI: `/platform/templates` listing grouped by kind; "Save as template" action contributed per-surface via a chrome/dialog-action region (same pattern as `rich-text.delete.actions`).

## Dependencies

- P-105 (builder eval) informs the `page`/`layout` kinds' payload shape but the registry itself is builder-agnostic.
