+++
id = "P-108"
title = "Components registry: reusable parametrized Puck blocks with Site sidebar item"
status = "open"
created = "2026-11-20"
priority = "med"
links = ["P-105", "P-107"]
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
