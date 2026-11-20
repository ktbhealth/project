+++
id = "P-108"
title = "Components registry: reusable parametrized Puck blocks with Site sidebar item"
status = "open"
created = "2026-11-20"
priority = "med"
links = ["P-102", "P-105", "P-107"]
epic = "site-builder"
+++

## Vision

A first-class **Components** concept for the site builder: reusable, parametrized blocks (Hero, RichText w/ Plate, CTA, DataTable, …) contributed by plugins or sourced from `@ktbhealth/ui`, usable inside pages, layouts, and templates. Puck's component model (typed `fields` → props) is the mechanism — we adopt the *concept*, not necessarily the library internals.

## Why a sidebar item

- Admins need a browsable catalog: "what blocks can I drop?" — name, description, preview, params.
- `/platform/components` (Site group) lists registered components with live preview + prop schema docs.
- Components are the design-token fidelity checkpoint: every block must render through `@ktbhealth/ui` tokens.

## Shape (proposal)

- Registry contribution kind `builder.components` (or reuse Puck's config registry): `{ id, label, description, fields schema, render, category, preview? }`.
- Core set first: layout primitives (section/container/columns), content (heading, rich-text→Plate, image, CTA), data (metric, table).
- Plugin-contributed: e.g. `patients` could expose a "PatientCard" block; `consultations` a "ConsultSummary" block — same registry, domain blocks.
- Admin page is read-only catalog + "insert into page/layout" affordance later.

## Open questions

- Build on Puck's component config directly vs own registry that compiles to Puck config (own registry keeps us editor-agnostic if we ever swap builders).
- Where do component definitions live: `@ktbhealth/ui` (design system) vs product plugins (domain blocks)? Likely split: primitives in ui, domain blocks in plugins.

## Relation to P-102 (done)

P-102 phase 3's "element-kind registry" (`{kind, label, editor, renderer, hooks}`) is the ancestor of this proposal. P-108 narrows it for the builder context: elements become *parametrized components* with typed props (Puck fields model) rather than free documents, and gains the `/platform/components` catalog. If the elements registry lands first, this proposal should extend it rather than fork it.

## Scope correction (2026-11-19)

`rich-text` is NOT part of this block set — `plugins/rich-text` already owns its `elementKinds` contribution (`kind: "rich-text"`, Plate editor) and stays a standalone plugin. `image` likewise lives in its own `plugins/image` (P-111), not in builder-blocks. The palette composes kinds from the `ElementKinds` registry — builder-blocks only ships true layout/content primitives (section, columns, heading, cta, metric, table).
