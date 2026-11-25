+++
id = "P-111"
title = "Image plugin: media library + image element kind for the site builder"
status = "done"
created = "2026-11-20"
priority = "med"
links = ["P-108", "P-109"]
epic = "site-builder"
+++

## Vision

`plugins/image` — the media layer for site content: an admin media library (`/platform/images`, Site group) plus an `image` element kind that builder blocks (P-108) and rich-text embeds can reference. Sibling to `plugins/rich-text` — rich text stays its own plugin; image is likewise its own concern, not folded into a generic components folder.

## Why its own plugin

- Media has lifecycle concerns blocks don't: upload, storage, derivatives (thumb/avif), alt text, deletion hooks (same `*DeleteHooks` pattern as rich-text/layouts/pages).
- Reusable beyond the builder: patient uploads, document-store assets, theme brand assets could converge later.
- Keeps `builder-blocks` focused on layout/content primitives; `image` and `rich-text` remain standalone element-kind citizens.

## Shape (proposal)

- `images` table (or document-store collection): `{ id, title, alt, mime, size, storageKey, width, height, createdAt }`.
- Storage: R2 in prod (theme Phase 4 already plans `THEME_ASSETS_QUEUE` + `/api/theme/assets` — evaluate sharing that pipeline vs dedicated binding); local/dev stub adapter.
- `elementKinds` contribution: `kind: "image"` — renderer = responsive `<img>`/`<picture>` through UI tokens; editor = picker dialog (library grid + upload) returning `{ imageId | url, alt, fit }` props.
- Admin page: grid library, upload, edit alt/title, delete (with referencer hooks so pages/layouts/rich-text get cleanup warnings like existing delete actions).
- Serve route: `/api/images/:id` or public asset URL with caching headers.

## Open questions

- Share `theme_assets`/document-store infra vs dedicated `images` table — decide after checking document-store collection fit.
- Derivatives now (thumb/avif via queue) or phase 2 — recommend phase 2, ship upload+serve first.
- Does the builder image block live in `builder-blocks` (thin wrapper over the `image` kind) or does `image` contribute its own block? Recommend: `image` plugin owns the kind; builder palette lists it automatically.
