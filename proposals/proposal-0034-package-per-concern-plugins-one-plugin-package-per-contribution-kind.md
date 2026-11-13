+++
id = "P-034"
title = "Package-per-concern plugins: one plugin package per contribution kind"
status = "deferred"
created = "2026-11-09"
owner = "devin"
priority = "med"
labels = ["architecture-platform"]
epic = "platform"
+++

## Context

The kernel contract already treats each contribution as its own plugin
*object* (`patients.tables`, `patients.services`, `patients.routes`,
`patients.landing-nav`, `patients.panel` — single-concern union,
deps per object). What is NOT separate today is the *package*: one
`plugins/<domain>` package exports all of a domain's concern objects.

The ask: make each concern literally its own plugin package —
`tables` is its own plugin, same for `services`, `routes`,
`landing-nav`, etc. — maximally composable and decoupled.

## Proposal

Flat package-per-concern layout (the discovery glob
`plugins/*/src/index.ts` is one level deep, so nesting like
`plugins/patients/tables` would NOT be discovered — packages must be
flat siblings):

```
plugins/
  patients-core/          # schema + repo helpers + types (shared lib)
  patients-tables/        # patients.tables plugin object
  patients-services/      # patients.services (deps: patients-tables?, platform.admin)
  patients-routes/        # patients.routes
  patients-landing-nav/   # patients.landing-nav
  patients-panel/         # patients.panel
```

Naming: `<domain>-<concern>`; concern suffix matches the plugin kind
(`-tables`, `-services`, `-routes`, `-landing-nav`, `-landing`,
`-landing-footer`, `-panel`, or bare for marker plugins).

## What changes vs. today

- **Deps become real package edges**: `patients-services` imports
  `@ktb/patients-tables/schema` (or a `-core` package) — package.json
  deps mirror `requiredDeps`. Today schema imports are implicit.
- **Independent testing**: each package tests only its concern.
- **Discovery unchanged**: `plugins/*/src/index.ts` still matches;
  each package exports `plugins: [oneObject]` (or a few same-kind
  objects).
- **Ids unchanged**: `patients.tables` etc. stay — the PluginRegistry
  doesn't care about packaging.

## Honest trade-offs

- **Package explosion**: ~6 domains × ~5 concerns ≈ 30 packages.
  tsconfig/vitest/package.json boilerplate per package; workspace
  install graph grows.
- **Shared internals need a home**: schema, repo helpers, view
  components, types are used by multiple concern packages of the same
  domain → a `<domain>-core` (or keep them in `-services`/`-views`
  and let others depend on it). This reintroduces a "kitchen-sink"
  package unless carefully split (schema+views+types = core;
  services = runtime).
- **The plugin OBJECT is already the unit of composition** — deps,
  topo order, registry, contributions all key off it, not the
  package. Package-per-concern buys file-level separation and
  independent versioning, not new runtime semantics.
- Alternative worth stating: keep package-per-domain (status quo) —
  the contract already gives per-concern decoupling; packages are
  just code organization.

## Recommendation

Pilot on the smallest domain (`users` or `groups` — services+panel
only) to measure boilerplate vs. gain before committing to the
30-package layout. If the pilot shows the `-core` package becomes a
junk drawer, the design fails its own decoupling goal.

## Open questions

- Does `<domain>-core` hold schema+views+types, or should views live
  with `-routes`/`-panel` (the consumers)?
- Should concern packages be allowed to export MULTIPLE same-kind
  objects (e.g. two panel entries) or strictly one?
- Migration order: new domains start split, or retrofit existing?
