+++
id = "P-112"
title = "Migrate content pages to the site page system ( phased: app → platform content → domain last )"
status = "open"
created = "2026-11-20"
priority = "med"
links = ["P-102", "P-107", "P-109", "P-111"]
epic = "site-builder"
+++

## Vision

Migrate existing pages onto the new page system (`pages` table + `builderJson` + element kinds) in phases, so the platform converges on one content model instead of three (file/contributed pages, panel.pages, DB custom pages).

## Inventory (product repo)

- **App-context `pages` contributions** (~9, e.g. `plugins/audience` landing via `landing:` kind) — content pages, best candidates.
- **`/platform/*` console pages** (~40: audit, i18n, kv, rbac, document-store, eval…) — these are *functional admin tools* (tables, forms, settings), not content.
- **Domain pages** (consultations, patients, agenda, clinical portals) — team actively working on them.

## Recommendation (honest scope cut)

- **Phase A — app/marketing content pages**: landing, legal/disclaimer, informational pages → `pages` rows (builderJson or template-instantiated). Clear win.
- **Phase B — platform**: only *content* surfaces migrate (e.g. dashboard widgets? docs links?). The ~40 admin consoles should **stay code** — they are applications, not documents; forcing them into a visual builder adds indirection with no benefit. Flagging this explicitly: "platform etc" in the original ask likely meant content pages, not CRUD consoles.
- **Phase C — domain pages (LAST)**: only after domain work settles. Scope = patient/doctor-facing *informational* fragments where a page system fits; core consult/patient UIs remain code permanently — they are workflows, not content.

## Shape

- Each migrated page becomes a `pages` row seeded from a template (P-107) or builder doc; contributed `pages`/`landing` descriptors get deprecated as rows take over.
- Domain element kinds (PatientCard etc.) live in their plugins — migration consumes them via `ElementKinds`, never forks them.
- Route keys (P-106) keep old paths stable during migration — no URL breakage.

## Open questions

- Which app pages are actually content vs functional? Needs a per-page audit (list the 9 app-context contributions).
- Does `landing:` contribution kind get absorbed by `pages` rows, or stay as the marketing-shell mechanism?
