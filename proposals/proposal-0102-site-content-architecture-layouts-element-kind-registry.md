+++
id = "P-102"
title = "Site content architecture: layouts + element-kind registry"
status = "open"
created = "2026-11-19"
priority = "med"
+++

## Context

Rich-text documents, custom pages, theme and (future) layouts/templates are converging into a "Site" content layer in the platform console. Sidebar group renamed Content → Site (phase 1, shipped separately). This proposal covers the two structural follow-ups.

## Phase 2 — Layouts (page-body composition)

- `layouts` table: `id, name, config JSON` — ordered sections with named content slots.
- `custom_pages.layoutId` FK (nullable → single-doc render, today's behavior).
- Layouts compose the page BODY only — NOT app chrome. Rationale: shells own chrome regions and cannot see which page/layout is active without plumbing; body composition sidesteps that entirely.
- Chrome remains the region model for shell slots (unchanged); a layout is a data preset over body sections.
- Referencers: layouts referencing rich-text docs register through the existing `RichTextDeleteHooks` pattern (same as pages).

## Phase 3 — Elements/Templates (kind generalization)

- Element-kind registry mirroring plugin concerns: plugins register `{kind, label, editor, renderer, hooks}` — rich-text becomes kind `"rich-text"`, the first citizen rather than a special case.
- Storage: one `elements` table (`kind` + `value` JSON) or per-kind tables behind the registry — decide when the second kind exists.
- Referencers via the hooks pattern, generalized from `RichTextDeleteHooks` to a document-agnostic contract.
- Templates = saved element graphs; pages instantiate a template and fill slots.

## Open questions

- Layout slot types: rich-text only, or arbitrary elements once Phase 3 lands?
- Do layouts belong to pages only, or also portal/marketing shells later?
- Migration path: existing pages keep working with `layoutId = null`.
