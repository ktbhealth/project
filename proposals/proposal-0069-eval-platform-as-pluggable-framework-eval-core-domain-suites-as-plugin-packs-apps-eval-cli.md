+++
id = "P-069"
title = "Eval platform as pluggable framework: eval-core + domain suites as plugin packs + apps/eval CLI"
status = "deferred"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["T-042", "T-118"]
+++

## Problem

`packages/eval-dataset` mixes ~60% generic machinery (runBatch, ports, RunRecord, report, calibration/accept gates, diff) with transcription-specific domain (TtsPort/AsrPort, WER/CER, personas, scriptgen, pipeline stages). Every new eval type would fork the package. The plugin system already solves "domains contribute capabilities with declared deps" — eval should reuse that shape.

## Proposal

```
packages/eval-core/                 infra below plugins — suite contract,
                                    batch/diff/report/gates, generic ports
                                    (TaskEnvelope, EvalPort, store/report),
                                    suite registry API
plugins/eval.<domain>/              domain packs — e.g. eval.transcription:
                                    Sample schema, metrics, stage pipeline,
                                    capability ports it needs, templates,
                                    persona matrix
apps/eval/                          @effect/cli composition root —
                                    discovers registered suites, composes
                                    layers per suite, exposes
                                    generate|run|gate|report|bench
```

- **Suite = manifest + stage functions**, not class hierarchy: `id`, `inputSchema`, `stages`, `metrics`, `gatePolicy`, `requiredPorts`.
- **Ports split by ownership**: generic ports in core; capability ports (Tts/Asr) in the suite that needs them.
- **Consultation is just a domain** — `plugins/eval.transcription` owns everything consult-specific; core knows nothing about transcripts.
- **apps/eval CLI** kills per-script arg-parsing duplication; adding a suite = adding a plugin dir, zero CLI changes.

## Caveat / dependency

The plugin registry today is UI/app-shaped (routes/nav/panel/services contributions). Eval suites need a new contribution kind (`eval.suites`) or the registry must be generalized first — see linked proposal on decoupling the registry from app-shaped contributions.

## Migration (incremental, no big-bang)

1. Extract `eval-core` package boundary (move generic files; re-exports keep imports stable).
2. Move transcription domain to `plugins/eval.transcription` (or `packages/eval-suite-transcription` if registry reuse slips).
3. Build `apps/eval` CLI absorbing `scripts/*.ts`.

## Acceptance

- A second eval suite can be added as a new plugin dir with no core changes.
- `apps/eval` lists/runs/gates any registered suite.
- All existing eval tests pass throughout migration.
