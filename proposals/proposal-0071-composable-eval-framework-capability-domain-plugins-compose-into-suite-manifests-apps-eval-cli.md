+++
id = "P-071"
title = "Composable eval framework: capability + domain plugins compose into suite manifests; apps/eval CLI"
status = "deferred"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["T-042", "T-118", "P-069", "P-070"]
+++

Supersedes P-069. Same goal, sharper layering: evals are **compositions of registered capabilities**, not monolithic suite packages.

## Layers

```
Layer 0  packages/eval-core/
         suite contract, runBatch, RunRecord/report, gates,
         suite registry API, generic ports — knows nothing
         about transcripts OR consultations

Layer 1  capability plugins — plugins/eval.cap.*
         eval.cap.asr      AsrPort contract + adapters
                           (stub / faster-whisper / Workers AI)
         eval.cap.tts      TtsPort contract + adapters
                           (stub / edge-tts)
         eval.cap.llm      EvalPort.judge (ACP / Jev)
         eval.cap.metrics  WER, CER, entity-match, negation
         eval.cap.store    DatasetStorePort (sqlite / R2 / mem)

Layer 2  domain plugins — plugins/eval.dom.*
         eval.dom.consult  Sample schema, personas, Eta
                           templates, scriptgen, PT-BR
                           LocaleProfile, consult gate policy

Layer 3  suite manifests — data, not code
         eval.suite.consult-asr = {
           domain:   eval.dom.consult
           caps:     [eval.cap.asr, eval.cap.tts, eval.cap.llm,
                      eval.cap.metrics, eval.cap.store]
           pipeline: synthesize → transcribe → correct → judge
           gates:    [wer-floor, entity-precision, negation, phi]
         }
         future suites (triage, extraction, dictation-asr…)
         = different domain/caps/pipeline/gates, zero new code
```

## Why composable > monolithic suite

- A suite is a **manifest composing registered capabilities** — `consult-asr` is config, not a package. Same caps serve `consult-asr`, `phone-intake-asr`, `dictation-asr`; only the domain pack differs.
- Capability plugins are suite-agnostic: `eval.cap.asr` doesn't know it serves consultations.
- `apps/eval` stays dumb: `eval run consult-asr --adapter local` → resolve manifest → collect required caps → wire layers → run. New suite = manifest + maybe a domain pack; new capability = one plugin.
- Maps onto P-070's generalized registry: caps register ports/adapters, domains register schemas/templates, suites register `eval.suites` contributions.

## Migration (incremental)

1. Extract `eval-core` (generic files move; re-exports keep imports stable).
2. Split today's domain into `eval.cap.*` + `eval.dom.consult` — more packages than P-069, better reuse.
3. Suite manifests + `apps/eval` CLI absorbing `scripts/*.ts`.

## Acceptance

- A second suite registers/runs with no core changes.
- Capability plugins are consumable by multiple suites.
- `apps/eval` lists/runs/gates any registered suite.
- All existing eval tests stay green throughout.
