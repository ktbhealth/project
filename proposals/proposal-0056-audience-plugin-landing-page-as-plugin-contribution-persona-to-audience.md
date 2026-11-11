+++
id = "P-056"
title = "Audience plugin: landing page as plugin contribution (persona to audience)"
status = "open"
created = "2026-11-11"
owner = "devin"
priority = "med"
labels = ["landing"]
links = ["P-035"]
epic = "platform"
+++

## Context

The landing page's audience picker (`apps/web/src/lib/persona.ts` +
`persona-select.tsx`, `?persona=doctors|patients|developers|business`)
is app-local glue: Paraglide labels, TanStack search params, docs-site
links. It cannot move to `@ktbhealth/ui` (no i18n/router there) and
doesnt fit `plugins/`'s current contract — there is no "landing
header" slot, and adding one would couple the host chrome to a feature.

Per P-035 (package-per-domain, concern-level plugin objects) and the
decoupling direction of P-043/P-044, the audience concept should be a
self-contained plugin that *is* the landing page, not a widget injected
into host chrome.

## Proposal

New `plugins/audience` package contributing `landing.page` — the
existing contract already lets a plugin own `/` (`index.tsx` resolves
`plugins.find(p => p.landing)?.page ?? Home`). The whole
persona-aware landing moves into the plugin:

- `AUDIENCES` (rename from PERSONAS — avoids collision with the Rive
  `Persona` in `@ktb/ui` ai-elements), parse/validate helpers, docs
  links (`DOCS_PATHS`, `docsHref`).
- `AudienceSelect` rendered inside the page's own chrome; `__root`
  drops `PersonaSelect`.
- Plugin i18n catalog (`audienceMessages(locale)`) following the
  `plugins/dashboard/src/i18n.ts` pattern — no Paraglide import.
- `?persona=` → `?audience=`; `validateSearch` stays in the host route
  (search schema is route-level; the plugin reads via `useSearch`).

## Kernel change (minimal)

`LandingPlugin.page` is `ComponentType` with no props — plugins can't
receive the active locale. Widen to `ComponentType<{ locale?: string }>`
(or reuse `PanelPageContext`), mirroring how `PanelHeaderAction` and
panel pages already get `locale` injected.

## Honest trade-offs

- **The real cost is copy porting**: `routes/index.tsx` is ~520 lines
  of `m.*` Paraglide strings → plugin catalog in en + pt-BR. Mechanical
  but large diff.
- `landing.page` is first-wins — audience becomes THE landing plugin;
  future landing experiments must compose inside it or wait.
- Reusability is architectural (consistent plugin model), not
  cross-repo: `website-draft` still cant consume `plugins/` — that
  would need `ui`.

## Acceptance

- `plugins/audience` registers `landing.page`; `/` renders it.
- `__root` has no audience/persona imports; `lib/persona*` deleted.
- Locale reaches the plugin page; catalog covers all current copy.
- `bun run check` + scoped vitest green in product and kernel.
