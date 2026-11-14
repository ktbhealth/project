+++
id = "P-070"
title = "Decouple plugin registry from UI/app-shaped contributions — generalize contribution kinds for non-UI domains"
status = "open"
created = "2026-11-14"
priority = "med"
links = ["P-069"]
+++

## Problem

The plugin registry (`plugins/*` manifests with `requiredDeps`/`optionalDeps`) was designed for app-shaped contributions: tables, services, routes, nav, chrome, pages, panel. Every contribution kind assumes a UI/runtime consumer. Non-UI domains — eval suites (P-069), infra tooling, batch jobs — have no clean way to register capabilities: they'd either fake UI contributions or bypass the registry entirely, losing dep-ordering and discovery.

## Proposal

Generalize the registry so contribution kinds are pluggable, not a fixed UI enum:

- **Contribution-kind registry**: kinds become declarative — each kind declares its slot name, payload schema, consumer(s), and ordering rules. Existing kinds (routes, services, tables, nav, chrome, pages, panel) become the first registered kinds; new kinds (`eval.suites`, `jobs.handlers`, `cli.commands`, …) register the same way.
- **Consumers opt in**: the app host collects kinds it knows; other consumers (apps/eval CLI, workers, scripts) collect their own kinds from the same plugin graph — one registry, many readers.
- **Dep graph stays kind-agnostic**: `requiredDeps`/`optionalDeps` already target concern-level plugin ids — keep that; only the contribution payload contracts get generalized.
- **packages/ vs plugins/ rule unchanged**: infra stays in packages; anything needing registry contributions or graph deps is a plugin — eval suites become `plugins/eval.<domain>` contributing `eval.suites` entries.

## Why now

P-069 (eval framework) is the first concrete non-UI consumer. Doing the registry generalization first (or alongside) avoids bending eval into fake route/page contributions and sets the pattern for jobs/CLI/other headless domains.

## Scope sketch

1. Audit `tools/arch-policy.mjs` + app-host registry code: enumerate current contribution kinds and their consumers.
2. Introduce kind registry (kind = { name, schema, consumers[], order? }) — likely in `@ktb/app-host` or a new `@ktb/plugin-registry` package.
3. Migrate existing kinds onto it without behavior change.
4. Add `eval.suites` kind as the proving consumer (with P-069).

## Acceptance

- A plugin can contribute a non-UI capability kind consumed by a non-app process.
- Existing plugins migrate with zero behavior change; arch-policy + tests stay green.
- Documented kind-authoring guide for future domains.
