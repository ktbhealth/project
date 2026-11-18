+++
id = "P-095"
title = "Shared oxlint rule packages: eslint-plugin-* + eslint-config-arch in ktbhealth/workspace"
status = "open"
created = "2026-11-18"
owner = "devin"
priority = "med"
+++

## Problem

ktb-arch oxlint rules live in `product/tools/` and only protect product. Hardcoded Tailwind colors already regressed once (fixed via PR #143 + `no-hardcoded-colors`); the same drift exists in storybook/docs/ui which have no lint enforcement. Rules should be shared org tooling, versioned independently.

## Decision

Extract each `ktb-arch/*` rule into its own unscoped TS package `eslint-plugin-<rule>` under `packages/` in the `ktbhealth/workspace` repo, plus `eslint-config-arch` exporting presets (`arch.product()`, `arch.ui()`). Consumed via git-tag deps (`github:ktbhealth/workspace#v0.1.0@eslint-plugin-<name>`), flavor-style tags per package for independent versioning.

## Shape

- 9 rule packages: no-hardcoded-colors, provider-lockin, package-boundary, tenant-scope, plugin-contract, plugin-route-gate, plugin-api-route-gate, client-contract, driver-contract.
- Each: TS, `vp pack` → dist, standalone (shared constants inlined, no dep graph).
- `eslint-config-arch`: presets returning `{ jsPlugins, overrides }` — `arch.product()` (all rules + product globs) and `arch.ui()` (colors only).
- All rule packages share plugin namespace `arch` → IDs `arch/<rule>` if oxlint merges same-named jsPlugins; fallback = distinct names mapped inside the config package (internal detail).
- Tests per package via fake-context/RuleTester; root workspace becomes bun workspaces + vite-plus.

## Rollout

1. workspace: scaffold + 10 packages + tests → review → tag `v0.1.0@eslint-plugin-*` ×9 (+ config tag).
2. product PR: consume `eslint-config-arch` `arch.product()`, delete `tools/ktb-arch-plugin.js` + `arch-policy.mjs`.
3. storybook PR: `arch.ui()`; expect violations — warn-first or fix.
4. docs + ui PRs: add oxlint infra + `arch.ui()`.

## Points of attention

- Rule IDs change `ktb-arch/*` → `arch/*` (or per-pkg prefixes if merging fails) — product overrides rewritten once.
- docs/ui have no lint today — adding oxlint is new infra, biggest scope.
- storybook likely has hardcoded colors in stories — budget fix or warn rollout.
- oxlint same-name jsPlugins merging unverified — config package abstracts either outcome.
- workspace repo gains its first versioned artifacts (tags) — release discipline starts here.
- `vp pack` lib mode must be verified in vite-plus 1.5.x; fallback tsdown/plain JS.
