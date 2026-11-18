+++
id = "P-097"
title = "Eval platform unification + AI capability plugins (ai-provider, ai-model, ai-prompt)"
status = "done"
created = "2026-11-18"
owner = "devin"
priority = "high"
labels = ["architecture-platform", "eval"]
links = ["T-042", "T-120", "T-135", "T-155", "P-069", "P-076", "P-081"]
done = "2026-11-18"
+++

## Problem

The eval effort exists as **two parallel surfaces that share a domain
pack but not a data model**, and the AI capability layer (providers,
models, prompts) is scattered across plugins, adapter packages, and
hardcoded constants:

- **Offline eval framework + CLI** (`packages/eval-core`, `apps/eval`,
  `evals/consultation`, capability plugins `transcription.tts/asr`,
  `llm.eval`, `storage.dataset`, adapter packages local/cloud/stub) —
  mature: contribution registry, stage DAGs, cross-suite inputs,
  domain definitions, experiment tracking, dataset manifests, judge
  calibration (P-069…P-079).
- **In-platform eval lab** (`plugins/eval-persona` → synthetic
  patients/doctors + `eval_instance_item` corpus;
  `plugins/eval-validator` → review gate + `eval_run`/`eval_run_sample`
  + panels) — generates real `patients`/`doctors` rows (`synthetic=1`)
  but runs are a separate table model, executed by `EvalRunner`
  (P-081) which is bound **only** when `EVAL_RUNNER_DIR` +
  `EVAL_ASR_BASE_URL` are set — otherwise runs sit `queued` silently.

Gaps: metric/gate/report math is conceptually duplicated; no queue,
retry, cancellation, or timeout on in-app runs; cloud adapter profile
is partial (T-120); prompts are hardcoded (`PROMPT`,
`CLEANUP_SYSTEM_PROMPT`, eval `buildScriptPrompt`); models are a
column on `ai_endpoint` plus scattered literals; generated synthetic
consultations never become real `consultations` rows, so they never
exercise the production transcription/anamnesis pipeline; no CI eval
gates or model/prompt versioning (T-135); pt-BR noisy multi-specialty
dataset still missing (T-042); `packages/eval-dataset/` and
`plugins/eval-credentials/` are empty stub dirs; eval deps may be
leaking into the prod Worker bundle (T-155).

## Proposal

One eval platform, two surfaces. The platform lab becomes a **driver**
of the same eval-core contracts; the CLI stays the headless/batch
surface. On top, three generic AI capability plugins give every
consumer (anamnesis, eval, consultations, future agents) a uniform,
versioned way to reach providers, models, and prompts.

### Decisions

- **D1 — Single contract, two stores.** `eval-core` owns all
  metric/gate/report/diff math. Platform maps `eval_instance_item` →
  dataset samples and `eval_run`/`eval_run_sample` →
  `Experiment`/`RunRecord` (shared manifest formats); the platform DB
  is the live-run store, the workdir/R2 store remains the CLI
  artifact store. `eval report`/`eval gate` consume both.
- **D2 — `ai-endpoints` evolves into `ai-provider`.** Rename (plugin
  id `ai.provider`, package `@ktb/ai-provider`), keep the
  `ai_endpoint`/`ai_agent_link` tables and ACP bridge machinery —
  it is already ~80% of the provider CRUD the org needs. No fork.
- **D3 — `ai-model` is the model catalog.** Per-provider model
  registry: `{providerId, modelId, kind: chat|vision|asr|tts|judge,
  contextWindow, capabilities[], enabled, isDefault}`. Adapters and
  plugins resolve models through it; the `model` column on
  `ai_endpoint` and hardcoded `@cf/*` literals migrate to catalog
  refs. Feeds `Experiment` model-version capture (T-135).
- **D4 — `ai-prompt` is a versioned prompt registry.** Prompts become
  data: `{name, version, scope, locale, template, variablesSchema,
  modelHint, status: draft|active|retired}` behind a `PromptPort`
  service. Hardcoded constants ship as seeded v1 prompts; consumers
  (`anamnesis-llm` prompt/cleanup, eval scriptgen/judge, consultations)
  resolve through the port. Prompt+model versions land in
  `Experiment` — the reproducibility fingerprint T-135 asks for.
- **D5 — In-app runs go through `@ktb/jobs`.** EvalRunner becomes a
  job consumer: durable enqueue, retries with backoff, per-sample
  timeout, cancellation, concurrency bound, and a visible
  "no runner bound" state instead of silent `queued`.
- **D6 — Synthetic consults enter the real pipeline.** Generation
  writes a `consultations` row per item (`synthetic=1` + provenance →
  `eval_instance_item.id`), so audio flows through the actual
  transcription → cleanup → anamnesis path; eval measures what
  production does, not a side channel.
- **D7 — Honest capability reporting.** `eval list`/`eval run`
  validate suite `caps` against the selected profile up front and
  name exactly what's missing (fixes T-120's silent early failure);
  the `cloud` profile gets real adapters or documented exclusions.

### Plugin quality bar (added to AGENTS.md)

Every new plugin is held to: **composable** (declared deps, no
import cycles), **modular** (port/service/adapter split), **testable**
(unit + contract tests, no untestable edges), **cacheable** (explicit
cache policy where reads repeat), **configurable** (env/schema-driven,
no hardcoded endpoints/models), **resilient** (timeouts, retries,
cancellation, degraded-mode fallbacks), **performant** (bounded
queries, no N+1, lazy heavy deps), **production-ready** (RBAC, i18n,
audit where sensitive, observability hooks). Applies retroactively as
a review checklist.

## Workstreams

- **WS-A `ai-provider`** — rename `ai-endpoints` → `ai-provider`
  (package, plugin id `ai.provider`, RBAC resource, routes, i18n
  keys); keep tables/bridge/links; CRUD UI already exists — polish to
  the quality bar (timeout/retry surfaced, health check, secretRef
  validation).
- **WS-B `ai-model`** — new plugin: `ai_model` table + CRUD UI +
  `ModelCatalog` port; seed known catalogs per provider kind; migrate
  `ai_endpoint.model` and adapter literals to catalog refs.
- **WS-C `ai-prompt`** — new plugin: `ai_prompt`/`ai_prompt_version`
  tables + CRUD/version UI + `PromptPort`; port `buildPrompt`,
  `CLEANUP_SYSTEM_PROMPT`, eval `buildScriptPrompt` into seeded
  prompts; wire version refs into eval runs.
- **WS-D eval data unification** — dataset-sample ↔
  `eval_instance_item` mapping; `eval_run` → `Experiment`/
  `RunRecord` emission; shared report/diff consumption in platform UI.
- **WS-E runner resilience** — jobs-queue consumer for EvalRunner;
  retry/backoff/timeout/cancel; concurrency bound; R2 artifacts in
  prod (`eval-adapter-cloud` already has r2); explicit unbound-runner
  UX.
- **WS-F adapter matrix** — cloud `llm`/`eval`/`tts` adapters or
  documented exclusions; profile/caps preflight diagnostics (T-120);
  optional `platform` profile (queue + cloud ASR/judge semantics).
- **WS-G synthetic pipeline** — `consultations.synthetic` +
  provenance column; generation writes consult rows + audio through
  the real pipeline; `@ktb/seed` patterns for baseline clinical
  entities; cascade cleanup story.
- **WS-H eval UI** — extend eval-persona/eval-validator panels into a
  full surface: run list, per-sample drill-down (audio +
  original/corrected transcript + verdicts), metric dashboards,
  experiment compare — via existing panel/`view.def` patterns, no new
  UI framework.
- **WS-I CI + versioning** — `eval gate --enforce` (stub profile in
  CI, local nightly); model+prompt+manifest versions in `Experiment`;
  pt-BR multi-specialty noisy pinned `DatasetManifest` (T-042 →
  T-135).
- **WS-J hygiene** — resolve empty `packages/eval-dataset` /
  `plugins/eval-credentials` (implement or delete); AGENTS.md quality
  bar; docs (`evals/README.md` + platform docs) reflect the unified
  model; check T-155 bundle leakage from eval deps.

## Non-goals

- No diagnosis/treatment/recommendation copy (positioning invariant).
- No public/unauthenticated eval API surface.
- No rewrite of eval-core contracts — extensions only.
- No new UI framework; panels/`view.def`/`document.collection` only.

## Risks

- **Rename churn** — `ai-endpoints` is referenced by eval-persona,
  eval-validator, credentials-adjacent code; mitigate with a
  compatibility re-export window.
- **Prompt injection via DB** — prompts become admin-editable data;
  mitigate with RBAC + versioning + review states + audit.
- **Bundle budget (T-155)** — eval/transcription deps in the prod
  Worker; WS-J must verify before WS-H ships UI.
- **Dual-store drift** — platform DB vs workdir/R2 artifacts; D1's
  shared manifest format is the mitigation, format tests are required.
- **PHI** — synthetic consults carry realistic PHI-shaped data;
  `synthetic=1` must gate every export/anonymize/anamnesis path.

## Acceptance criteria

- One metric/gate/report implementation consumed by CLI and platform.
- `eval list` names missing caps per profile; `cloud` profile either
  complete or explicitly partial with diagnostics.
- In-app eval runs survive process restart (jobs queue), retry failed
  samples, and cancel cleanly.
- A generated synthetic consult appears in the doctor UI as a normal
  consultation and flows through transcription → cleanup → anamnesis.
- Prompts/models/providers are CRUD-managed; eval `Experiment` records
  their versions; `eval gate --enforce` runs in CI.
- Empty stub dirs resolved; AGENTS.md carries the plugin quality bar.

## Suggested order

WS-A → WS-B → WS-C (provider/model/prompt trio, each independently
shippable) → WS-D → WS-E/WS-F → WS-G → WS-H → WS-I → WS-J continuous.
WS-C unblocks prompt versioning early; WS-G depends on WS-E for
throughput.

## Links

T-042, T-120, T-155, T-135 · P-069…P-079 (framework lineage) · P-081
(in-app validator) · ADR-0001 (transcription sources)
