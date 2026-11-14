+++
id = "P-074"
title = "Composable eval framework v4: hardened manifests, versioned runs, registry fallback path"
status = "in-progress"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["T-042", "T-118", "P-070", "P-071", "P-072", "P-073"]
+++

Supersedes P-073. Same architecture; adds the hardening gaps found in design review.

## Architecture (unchanged from P-073)

```
packages/eval-core/       contracts + orchestration ONLY
                          (suite schema, runner, gates engine,
                          report/diff, generic ports)
plugins/transcription.*  product capabilities + stage contributions
plugins/llm.*            EvalPort.judge + stage.judge
plugins/storage.dataset/ DatasetStorePort impls
plugins/eval.dom.*       eval-only domain packs (consult first)
evals/*.ts               typed suite manifests — CLI input, not
                         registry citizens
apps/eval/               @effect/cli — list|generate|run|gate|
                         report|bench
```

Manifest example, CLI usage, migration phases: see P-073 (identical).

## Hardening additions

1. **Manifests are pure data.** `defineSuite()` accepts a Schema-validated
   object ONLY — stages/metrics/gates/caps referenced by registered id,
   never inline functions. This keeps `evals/*.ts` diff-able and prevents
   logic creep; type-checking without executable content.

2. **Registry fallback path.** P-073 assumed P-070's generalized
   contribution kinds. Interim: eval-core ships its own mini-registry
   for stage/metric/gate contributions; when P-070 lands, those
   registrations migrate to the shared kind registry with zero manifest
   changes. `evals/` dir-scan + plugin-id resolution works regardless.

3. **Versioned runs for comparability.** Every manifest carries
   `schemaVersion`; every RunRecord pins the resolved contribution
   versions (plugin ids + adapter profile + manifest hash). `eval
   report`/`diff` can then trend runs across manifest evolution —
   required for calibration over time, not just single runs.

4. **Manifest resolution is CI-tested.** `apps/eval list` runs in CI;
   each manifest gets a `--dry-run` resolution smoke (deps resolve,
   schemas validate, no ports wired). A manifest that can't resolve
   fails the build, not the 3am run.

## Non-goals (explicit)

- No distributed queue / continuous eval — `runBatch` per-process
  concurrency is the right scope; revisit only if suites go continuous.
- No orchestrator perf work — eval is I/O-bound; optimize adapters.
- `evals/` manifests are not plugins and do not join the registry.

## Acceptance (P-073's + new)

- All P-073 acceptance criteria.
- `defineSuite` rejects non-data input at type level AND Schema level.
- RunRecords carry schemaVersion + resolved-version pins.
- CI fails on unresolvable manifests.
- Works end-to-end before AND after P-070 lands (fallback registry).
