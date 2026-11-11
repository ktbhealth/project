+++
id = "PL-003"
title = "GitHub Actions CI across org repos (T-030)"
status = "open"
created = "2026-11-09"
owner = "human:emanuel"
priority = "med"
labels = ["github-actions"]
+++

# Plan: GitHub Actions CI for the ktbhealth platform (T-030)

## Context
T-030 (owner human:emanuel): "Configure GitHub Actions CI; later integrate Peregrin as CI orchestrator inside GHA". Today only `repos/product` has a workflow (`.github/workflows/ci.yml`: bun install --frozen-lockfile, `bunx vp check`, test, build on push to main + PR). The other five JS repos have none. Goal: consistent CI across all repos, GHA-first so Peregrin (T-026) can wrap it later.

## Findings
- All repos pin `packageManager: bun@1.4.4`; `product` ci.yml uses `oven-sh/setup-bun@v2` + node 22.
- Scripts: storybook (check, lint, build), website-draft (check, lint, test, build), slidev-theme-ktb (test, build), presentation-draft (build), ui (none).
- storybook/website-draft deploy via `cuora run … alchemy deploy` (secrets) — out of scope for the first pass.
- Consumers depend on `github:ktbhealth/ui#vX.Y.Z` git tags: `bun install` fetches from GitHub; ui is public/private? (verify tokens needed in CI).

## Steps
1. **Record in myco** (first step after approval), from `repos/project`:
   `myco plan new "GitHub Actions CI across org repos" --owner human:emanuel --pri med --label github-actions --blocks T-026 --body <this plan>`
   then `myco handoff T-030` pointing at the new PL-id. Check `myco item set` / `myco item show T-030` for how to link PL-id to T-030 (no direct link verb seen; fall back to a handoff + label).
2. **Harden `product/.github/workflows/ci.yml`**: `permissions: contents: read`, `concurrency` (cancel-in-progress on PRs), bun cache, pin node via setup-node.
3. **New `ci.yml` in each repo** (same skeleton, per-repo steps):
   - storybook: `bunx vp check`, build
   - website-draft: `bunx vp check`, test, build
   - slidev-theme-ktb: test, build (`example.md`)
   - presentation-draft: build
   - ui: `bun install --frozen-lockfile` + `tsc` typecheck only if a script is added; otherwise install-only smoke job
4. **Deploy workflows (deferred)**: separate `deploy.yml` using cuora/alchemy, decided after secrets strategy.
5. **Peregrin (later, T-026)**: keep each job a single `bun run <script>` so Peregrin can orchestrate the same commands.

## Open questions
- Is `ktbhealth/ui` private? If so, CI needs a token/deploy key for `github:ktbhealth/ui#…` deps.
- Include deploy workflows now or later?

## Verification
- `actionlint` (or `gh workflow` view) on every new YAML.
- Open a PR per repo and confirm the check passes; run `bun install --frozen-lockfile && bun run <script>` locally first.
- `myco check` after the ledger write; `myco plan list` shows the new plan.
- No push/commit without explicit user approval.
