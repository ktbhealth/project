+++
id = "P-076"
title = "Eval platform market-parity: experiment tracking, dataset registry, judge calibration — data layer only"
status = "open"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["T-042", "P-074", "P-075"]
+++

Gap analysis vs market eval platforms (LangSmith, Braintrust, Promptfoo, DeepEval, Inspect AI). Our foundation (typed contracts, RunRecord audit, Wilson gates, PHI policy) is ahead; these are the missing capabilities — all data/model layer, NO UI required yet.

## 1. Experiment tracking / run versioning

Market tools treat each run as an experiment with comparable metadata. We have RunRecords + JSONL but no experiment identity.

- `Experiment` = {id, suiteId, manifestHash, gitSha, profile, dataset{id,version}, startedAt, summary}
- Store experiments in the DatasetStorePort (new `experiments` collection or sidecar index) — every `eval run` writes one
- `eval report --compare <expA> <expB>` — diff metrics/gates between experiments (data-only comparison, markdown output)
- Manifest `schemaVersion` + resolved-contribution pins (already in RunRecord plan) become the experiment's reproducibility fingerprint

## 2. Dataset registry

Today datasets are generated artifacts with no registry.

- `DatasetManifest` = {id, version, domain, generator params (matrix/seed/locale), provenance, createdAt, sampleCount, contentHash}
- `eval generate` writes a DatasetManifest alongside samples; store validates id+version uniqueness
- Suites reference `{id, version}` — already in manifest schema; add resolution check in `eval list --strict`
- Enables: dataset diffing (which personas/templates changed), regression pinning (suite pins dataset version)

## 3. Judge quality / calibration

Market judges are calibrated on corpora; ours are adapters with no quality tracking.

- `JudgeReport` per run: agreement stats vs gold where available, probability calibration buckets (already have confSegments — extend to judge probabilities), per-question-kind accuracy
- `eval calibrate <suite>` — run judge-vs-gold agreement over a labeled subset, emit calibration report; feeds threshold recommendations into gate policy (the M2 vision: confidence-aware auto-accept)
- Track judge model/version in RunRecord (systemone `model` field, jev version, acp endpoint hash) for drift detection

## 4. Generalization proof (the real market gap)

Market tools prove breadth by usage. Ours needs a second domain to validate the framework claim.

- Priority second suite candidates: `dictation-asr` (same caps, different domain pack — cheapest proof), `triage`, `extraction`
- Acceptance: second suite lands as `evals/<domain>/` + manifest with ZERO eval-core changes — this is the P-075 acid test

## 5. Integrations (deferred, noted)

- CI: `eval gate --enforce` already exits correctly; add `--format json` for CI parsers (cheap)
- Notifications/webhooks on gate failure — defer until a consumer exists
- OTLP/observability: RunRecords could emit spans — defer, observability package exists if needed later

## Non-goals

- No UI/dashboard — data layer + CLI output only
- No hosted SaaS, no external deps beyond existing adapters
- No prompt-template management (manifests cover it)

## Sequencing

1. P-075 lands (evals/<domain>/ restructure) — experiments/datasets reference domain packs
2. Experiment tracking + dataset registry (small: store schema + 2 commands)
3. Judge calibration report
4. Second suite as proof
