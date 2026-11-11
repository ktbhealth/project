+++
id = "PL-005"
title = "Plugify landing chrome: brand, CTA, session menu, footer note as plugin contributions"
status = "deferred"
created = "2026-11-11"
priority = "med"
labels = ["plugins"]
links = ["P-058", "P-060"]
epic = "platform"
+++

Prerequisite for full enable/disable coverage in P-058: the marketing chrome in `apps/web/src/routes/__root__.tsx` is still host-hardcoded. Nav links (`landingNav`) and footer blocks (`landingFooter`) are already plugin-driven; everything else must become plugin contributions first so the Plugins manager can toggle it.

## What's hardcoded today (RootDocument)

- Brand block: BookPlusIcon + "KTB Health" wordmark → `/`
- "New consultation" CTA → `/consultations/new` (belongs to consultations domain)
- LanguageToggle (paraglide locales)
- Auth chrome: sign-in link / user dropdown + sign-out (sessionFn data already in root loader)
- Footer disclaimer `m.footer_disclaimer()` (host i18n copy)
- `MissingLanding` fallback in `routes/index.tsx`

Stays host-owned regardless: `isConsole` pathname split, TooltipProvider, Scripts, devtools, error/404 components, session loader itself.

## Steps

1. **kernel contract** (`packages/kernel/src/contributions.ts`): new single-concern contribution kinds —
   - `landingBrand`: `{ to, label, icon? }` data (LandingLink-shaped; icon like PanelSidebarItem)
   - `landingHeaderActions`: `ComponentType<LandingChromeProps>[]` — arbitrary header widgets; `LandingChromeProps = { user, locale, plugins }` injected via a host `LandingChromeContext` (mirrors `PanelPageContext`)
   - footer note: extend `LandingFooterPlugin` with optional `note?: string` key, host-mapped through a `pluginFooterNote()` i18n helper (same mechanism as `pluginNavLabel`/`pluginPanelLabel`) — plugin declares an English/key string, host localizes
   - each new field gets `ForbiddenContributions` coverage + union member

2. **`LandingChromeContext`** in app-host client (or apps/web lib): `{ user: session | null, locale, plugins }` — root route already loads `sessionFn`; provider wraps non-console chrome.

3. **New/updated plugins:**
   - `plugins/consultations` → `landing.headerActions` CTA "New consultation" (or data-driven `landingNav` item with `cta` styling flag — pick component slot for flexibility)
   - `plugins/session` (new package) → sign-in link + user dropdown; imports `@ktb/auth-better/client` (package, allowed); reads `user` from context
   - `plugins/site` (new package, or fold into an existing chrome plugin) → `landingBrand` + `landingFooter.note`
   - locale toggle: either a `landing.headerActions` widget in `plugins/site` (needs `changeLocale` exposed via context — host i18n lib can't be imported by plugins) or stays host. **Decision needed.**

4. **Host rewrite** (`__root__.tsx`): render brand/nav/actions/session/footer slots from the filtered plugin list; delete hardcoded blocks. `index.tsx` `MissingLanding` → either keep as last-resort fallback or move into `plugins/site` as the default `landing.page` (it currently only renders when NO landing plugin exists — keep host fallback, it's the empty-registry case).

5. **Parity + tests**: `plugins/*/src/{index,client}.ts` parity assertion (`apps/web/src/plugins.test.ts`) covers new fields; kernel type tests for the union; `-root.test.tsx` updated for slot rendering; `bun run check` + scoped vitest green.

## Ordering vs P-058

Land this first (or alongside): P-058's enable/disable then covers chrome for free since filtering is per-contribution-kind. No contract conflict — P-058 adds `meta`/`config`, this adds contribution kinds.

## Open decisions

- Locale toggle: plugin widget (needs locale-switch fn injected via context) vs host-owned (paraglide is host runtime).
- Footer disclaimer: plugin `note` key vs host-owned legal copy.
- Session UI in `plugins/session` vs keeping auth chrome host-owned (auth is infra, not domain — but user wants max plugability).
- Brand as data (`landingBrand`) vs component slot — data preferred, matches LandingLink style.
