+++
id = "P-060"
title = "Chrome regions: shell-owned named slots (header/footer/sidebar) with ordered plugin contributions + when-rules"
status = "open"
created = "2026-11-11"
priority = "med"
labels = ["plugins"]
links = ["P-058", "P-059", "PL-005"]
epic = "platform"
+++

## Context

PL-005 plugifies the marketing chrome with ad-hoc contribution kinds (`landingBrand`, `landingHeaderActions`, footer `note`). This proposal generalizes that into a **region model**: named composable slots owned by shells, plugins contribute into them with order + per-page visibility. Supersedes the contract-design part of PL-005 (its migration steps fold in).

## Model

Shells own regions; plugins never hardcode position.

```
app shell (plugins/site)              console shell (platform-sidebar)
  app.header.brand                      console.sidebar
  app.header.nav                        console.header.actions
  app.header.actions
  app.header.locale
  app.footer.blocks
  app.footer.note
  app.sidebar                    ← new: per-page sidebars (e.g. consultations filters)
```

Contribution — one new single-concern field on `Plugin`:

```ts
chrome: [{
  region: string,              // must be a region the active shell declares
  id: string,
  content: ComponentType<ChromeProps> | LandingLink | LandingFooterBlock | …,
  order?: number,              // explicit order; falls back to plugin topo order
  when?: { paths?: string[]; exceptPaths?: string[] },  // declarative, host-evaluated
}]
```

`ChromeProps = { user, locale, plugins }` via a host `ChromeContext` (generalizes PL-005's `LandingChromeContext`, mirrors `PanelPageContext`). Data contributions (nav labels, footer note) keep the existing host i18n-mapping mechanism; component contributions resolve copy from `locale` like `panel.headerActions` today.

`when` rules are **data, not functions** — the client/server registries stay serializable-clean and rules are unit-testable. No match → slot doesn't render.

## Mapping of existing contributions

- `landingNav` items → `app.header.nav` region (LandingLink content)
- `landingBrand`, CTA, locale toggle, session menu → `app.header.*` (PL-005's planned plugins: `plugins/site` owns brand/locale/footer-note, `plugins/session` owns session menu, `consultations` owns the CTA)
- `landingFooter.blocks`/`note` → `app.footer.*`
- `panel.sidebar`/`panel.headerActions` → alias or migrate to `console.sidebar`/`console.header.actions` regions (same machinery both shells — decide alias vs migrate at implementation)

`plugins/site` is the app-shell plugin: it declares the app regions and contributes brand/locale/footer. Region *declaration* is shell-owned; plugins only fill slots.

## Explicitly deferred (backlog, not v2 scope here)

- Page-dependent region overrides (`panel.pages`/`pages` descriptor declaring `regions: { hide: [...] }`) — noted as future, parked.
- Arbitrary nesting/slot trees — flat list per region + `order` is enough. Deliberately not going Drupal.
- `app.sidebar` gets defined in the contract but no consumer required for v1.

## Integration

- Enable/disable (P-058): region filtering rides the same enabled-set filter as every other contribution — disabled plugin's chrome contributions vanish uniformly.
- `pages` contribution (P-059): independent; region model doesn't depend on it.
- Order of work: contract + `plugins/site` + migrate `__root__.tsx` → then P-058 manager covers chrome.

## Open questions

- `panel.*` → `console.*` regions: alias for compatibility or hard migration (update all panel plugins at once)?
- Region declaration site: does the shell plugin declare regions via a `regions` contribution field, or does the host hardcode known region names per shell? Lean: host declares the two shells' region sets (they're host chrome after all); plugins/site fills app regions.
- `order` vs topo-order precedence when both present.
- Unknown region name in a contribution → dev-time warning or hard error at discovery?
