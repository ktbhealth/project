+++
id = "PL-009"
title = "Eval platform unification + AI capability plugins — execution"
status = "done"
created = "2026-11-18"
owner = "devin"
priority = "high"
links = ["T-042", "T-120", "T-135", "T-155", "P-097"]
+++

Execution plan for P-097 — eval platform unification + AI capability
plugins. Each phase is independently shippable; WS letters match the
proposal's workstreams.

## Phase 1 — AI capability trio (foundation)

- [ ] WS-A: rename `plugins/ai-endpoints` → `ai-provider`
      (`@ktb/ai-provider`, plugin id `ai.provider`, RBAC `ai.provider`,
      compat re-exports during transition); polish CRUD to quality bar
      (health check, secretRef validation, timeout/retry surfaced).
- [ ] WS-B: `plugins/ai-model` — `ai_model` table, CRUD UI, `ModelCatalog`
      port, per-provider seeds, migrate `ai_endpoint.model` + adapter
      literals to catalog refs.
- [ ] WS-C: `plugins/ai-prompt` — `ai_prompt`/`ai_prompt_version`
      tables, versioned CRUD UI, `PromptPort`; port `buildPrompt`,
      `CLEANUP_SYSTEM_PROMPT`, eval `buildScriptPrompt` to seeded v1
      prompts; consumers resolve via port.
- [ ] AGENTS.md: add the plugin quality-bar section (composable,
      modular, testable, cacheable, configurable, resilient,
      performant, production-ready).

## Phase 2 — Eval data unification

- [ ] WS-D: `eval_instance_item` ↔ dataset-sample mapping;
      `eval_run`/`eval_run_sample` → `Experiment`/`RunRecord` emission;
      shared manifest format + format tests; platform UI consumes
      eval-core report/diff (no parallel metric math).

## Phase 3 — Runner + adapters

- [ ] WS-E: EvalRunner as `@ktb/jobs` consumer — durable enqueue,
      retry/backoff, per-sample timeout, cancel, concurrency bound,
      visible unbound-runner state; R2 artifacts in prod.
- [ ] WS-F: adapter matrix — cloud `llm`/`eval`/`tts` adapters or
      documented exclusions; `eval list`/`run` caps-vs-profile
      preflight with named gaps (T-120); optional `platform` profile.

## Phase 4 — Synthetic data + UI + CI

- [ ] WS-G: `consultations.synthetic` + provenance →
      `eval_instance_item`; generation writes real consult rows +
      audio through the production transcription → cleanup →
      anamnesis pipeline; `@ktb/seed` for baseline entities;
      cascade cleanup (T-042).
- [ ] WS-H: eval UI completion — run list, per-sample drill-down
      (audio + transcripts + verdicts), metric dashboards, experiment
      compare; panels/`view.def` only.
- [ ] WS-I: `eval gate --enforce` in CI (stub profile) + local
      nightly; model+prompt+manifest versions in `Experiment` (T-135);
      pinned pt-BR noisy `DatasetManifest`.
- [ ] WS-J: hygiene — empty `packages/eval-dataset` /
      `plugins/eval-credentials` resolved; T-155 bundle-leak check;
      docs updated (`evals/README.md`, platform docs).

## Ordering notes

WS-C early (prompt versioning feeds Experiment). WS-G depends on
WS-E (throughput/queue). WS-H depends on WS-D (shared report data).
WS-I last — needs stable version capture from WS-B/C.

## Open questions (from P-097)

- Platform DB vs workdir/R2 as canonical live-run store (D1 assumes
  shared manifest format either way).
- In-app runner in-Worker (cloud adapters) vs dev-only — decides
  WS-E/WS-F scope.
- `ai-endpoints` rename vs keep-and-wrap — WS-A decides at kickoff.

Links: P-097 · T-042 · T-120 · T-135 · T-155
