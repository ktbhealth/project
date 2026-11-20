+++
id = "P-105"
title = "Visual page builder for product marketing/admin pages (Puck + Plate eval)"
status = "done"
created = "2026-11-19"
owner = "human:Luis-Emidio"
priority = "med"
links = ["P-109"]
epic = "site-builder"
done = "2026-11-20"
+++

## Context

We may want an Elementor-like experience for building pages (landing pages, patient/doctor-facing content, admin-configured views) without shipping code per page. Two layers are needed:

- **Layout/page builder** — drag-drop sections, reorder, responsive preview, JSON document.
- **Rich text editing** — already solved: `platejs` powers `RichTextEditor` in `@ktbhealth/ui` (`packages/react/src/components/rich-text-editor.tsx`).

## Candidate: Puck

- MIT, ~14k stars, actively maintained; `<Puck>` embeds in React, pages serialize to JSON, `<Render>` outputs the page. We bring our own components → register `@ktbhealth/ui` (Base UI/shadcn-style) components so output is on-brand.
- Complements Plate: a Puck "RichText" component can render/embed Plate for content editing inside a section.
- Newer alternatives noted but unproven: ui-builder (shadcn-native, ~679★), crafted-design (Craft.js-based, claims 1.0).

## Open questions

- Which surface needs it first: public marketing site (`website-draft`), docs, or product in-app pages (plugins/P-094 dynamic platform overlap)?
- Is this a duplicate of P-094 "dynamic runtime platform" scope, or a distinct UI-layer capability?
- Persistence/render target: plugin-provided pages (P-104) vs product routes vs website-draft?
- Does Puck's field/component model map cleanly onto our plugin architecture and design tokens?

## Next step

Spike: embed Puck in a scratch route, register 3-4 `@ktbhealth/ui` components + a Plate-backed rich-text field, render a sample landing page. Evaluate DX, token fidelity, JSON schema stability, and bundle cost before committing.

## Platform mapping (product repo findings)

- `/platform` sidebar **Site group already exists**: Pages, Layouts, Rich text, Schemas, Fonts, Theme — all `portal.sidebar` contributions with `group: "Site"`.
- **Pages** (`plugins/pages`, T-083): `custom_pages` table — path, published, `showInNav`, optional `richTextId` / `layoutId` / `searchSchemaId`. Rendered by a `/*` wildcard page descriptor; admin CRUD at `/platform/pages`.
- **Layouts** (`plugins/layouts`): `layouts` table — ordered named sections, each slot binds a rich-text document. Pages compose their body via `layoutId`.
- **Templates**: no first-class entity today; closest analogs are reusable rich-text docs and layouts as page skeletons.

### Direction

- Puck is the **editor surface of Pages**, not a separate sidebar item — "edit page" opens the Puck canvas. Sidebar unchanged (Pages entry stays the entry point).
- **Additive data model**: new `builderJson` column on `custom_pages` holding the Puck document. `richTextId`/`layoutId` path keeps working for simple/legacy pages; builder pages render via Puck `<Render>`.
- **Templates** emerge later as saved Puck documents instantiated into new pages — no schema needed now.
- Layouts stay as-is initially; possible later convergence (Puck sections subsuming layout slots) is a separate migration.
