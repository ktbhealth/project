+++
id = "P-072"
title = "Composable eval framework v2: shared product capabilities + eval domain packs + evals/ suite manifests"
status = "open"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["T-042", "T-118", "P-069", "P-070", "P-071"]
+++

Supersedes P-071. P-071's flaw: capabilities lived in an eval namespace (`eval.cap.*`) — but ASR/TTS/LLM are **product capabilities**. Eval must measure the same adapters production runs, not forks of them.

## Layers

```
Layer 0  packages/eval-core/
         EvalSuite contract, runBatch, RunRecord/report sinks,
         gate engine, suite registry API, generic ports
         (TaskEnvelope, EvalPort judge contract, ReportPort)
         — knows nothing about transcripts OR consultations

Layer 1  capability plugins — NOT eval-owned
         Ordinary product plugins contribute ports + adapters;
         eval consumes the same registry the app does:
           plugins/transcription.asr    AsrPort + stub/local/cloud
           plugins/transcription.tts    TtsPort + stub/local
           plugins/llm.*                EvalPort.judge (ACP/Jev)
           plugins/storage.dataset      DatasetStorePort impls
           plugins/eval.metrics.*       WER/CER/entity metrics
         → eval measures production adapters; product
           transcription ships free; any new product
           capability is eval-able with zero eval code

Layer 2  eval domain packs — plugins/eval.dom.*
         eval.dom.consult  Sample schema, personas, Eta
                           templates, scriptgen, PT-BR
                           LocaleProfile, consult gate policy
         (domain packs are eval-only; capabilities are shared)

Layer 3  suite manifests — evals/ at repo root
         evals/consult-asr.yaml:
           domain:  eval.dom.consult
           dataset: consult-ptbr-v1
           stages:  [synthesize, transcribe, correct, judge]
           caps:    [transcription.asr@local,
                     transcription.tts@stub,
                     llm.jev@acp,
                     eval.metrics.wer,
                     storage.dataset@sqlite]
           gates:   [{id: wer-floor, min: 0.9}, ...]
           adapterProfile: local

         Stages, metrics and gates are themselves registered
         contributions — manifests order/select them, they are
         never hardcoded. A suite is pure composition data.

apps/eval/  @effect/cli — discovers evals/*, resolves the
             plugin graph, wires layers per suite, exposes
             list | generate | run | gate | report | bench
```

## Why this is the right shape

- **No eval/prod drift**: the ASR adapter under test IS the product's adapter — same plugin, same layer wiring.
- **Pipeline is data**: reordering/filtering stages, swapping metrics or gate thresholds = manifest diff, reviewable in PR.
- **evals/ is self-documenting**: every suite in the repo is visible at a glance; adding one is a YAML file + maybe a domain pack.
- **Domain packs reusable beyond eval**: personas/templates/scriptgen can feed demos and dev fixtures.
- Maps onto P-070: caps/domains/stages/gates/suites are all contribution kinds on the generalized registry.

## Migration (incremental)

1. Extract `eval-core` (generic files; re-exports keep imports stable).
2. Move transcription domain → `eval.dom.consult`; ASR/TTS/LLM/store adapters → product capability plugins (`transcription.*`, `llm.*`, `storage.dataset`) — eval packages become thin consumers.
3. `evals/` manifests + `apps/eval` CLI absorbing `scripts/*.ts`.

## Acceptance

- A second suite registers/runs with no core changes — often no new code at all.
- Eval adapters are literally the product adapters (same plugin ids).
- Changing a suite's stages/metrics/gates is a manifest edit.
- `apps/eval` lists/runs/gates any `evals/*` manifest.
- All existing eval tests stay green throughout.
