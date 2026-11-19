+++
id = "P-105"
title = "Visual page builder for product marketing/admin pages (Puck + Plate eval)"
status = "open"
created = "2026-11-19"
priority = "med"
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
