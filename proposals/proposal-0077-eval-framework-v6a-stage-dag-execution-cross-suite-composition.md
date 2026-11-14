+++
id = "P-077"
title = "Eval framework v6a: stage DAG execution + cross-suite composition"
status = "open"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["P-074", "P-075", "P-076"]
+++

Next composability ceiling after P-075/P-076. Two related execution-model upgrades.

## 1. Stage DAG (not just ordered subsets)

Current dispatch (PR #26): manifest `stages` must be a subsequence of canonical order — no parallelism, no conditionals, no fan-out.

Target: stages form a **DAG** — nodes are registered stage contributions, edges are declared data dependencies.

- Manifest declares `stages: [{id, needs: [...], optional: bool}]` or a `dag:` section; eval-core topologically sorts, runs independent branches concurrently (bounded by runBatch concurrency), merges RunRecords per branch.
- Enables: `transcribe ∥ secondary-asr` (compare two ASR engines in one run), conditional stages (`judge` only `when: wer > threshold` — condition evaluators are registered contributions, serializable params only), fan-in metrics.
- RunRecord gains `branch`/`parent` fields for DAG provenance; gate engine evaluates per-branch or merged metrics (policy decides).
- Constraint: keep deterministic ordering where the domain requires it (consult pipeline is inherently sequential — DAG is opt-in per suite).

## 2. Cross-suite composition

Suites currently run independently. Enable suite outputs as inputs:

- `evals/<domain>/<suite>.suite.ts` gains `inputs:` — references to other suites' persisted artifacts (e.g. triage suite consumes consultation suite's corrected transcripts; extraction suite consumes ASR output).
- `DatasetStorePort` already persists SampleResults — add `SuiteOutputRef` resolution: a suite can declare `from: {suite, stage, field}` and the runner materializes inputs from prior runs (same dataset version or explicit run id).
- Experiment records (P-076) become the lineage: experiment B records `derivedFrom: experimentA`.
- Lineage is data — manifests declare it, runner resolves it, RunRecords carry it.

## Why now

P-075 makes domains data-owned; P-076 adds experiment/dataset identity — both are prerequisites. DAG + composition are the last structural ceilings; after this the framework is done growing (runtime-loaded plugins remain backlog).

## Acceptance

- A suite can declare parallel stage branches and they run concurrently with merged audit records.
- Conditional stage execution via registered, serializable condition contributions.
- A suite can consume another suite's stage output as input, with lineage recorded.
- Existing linear suites unchanged.
