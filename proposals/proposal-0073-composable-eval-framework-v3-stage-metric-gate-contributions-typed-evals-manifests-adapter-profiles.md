+++
id = "P-073"
title = "Composable eval framework v3: stage/metric/gate contributions, typed evals/ manifests, adapter profiles"
status = "open"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["T-042", "T-118", "P-070", "P-071", "P-072"]
+++

Supersedes P-072. Review fixes: metrics belong to capability/domain owners (not eval plugins); stages are contributed by the plugins that own them; manifests are typed TS validated by eval-core Schema (not YAML); evals/ is CLI input, not registry citizens; capability id and adapter profile are separate decisions.

## Final architecture

```
packages/eval-core/            contracts + orchestration ONLY
  suite.ts    EvalSuiteManifest schema; stage/gate/metric
              contribution types
  runner.ts   stage executor, runBatch, RunRecord
  report.ts diff.ts            sinks + reversible diffs
  gates.ts                     policy-agnostic gate engine
  ports.ts                     TaskEnvelope, EvalPort(judge),
                               ReportPort, generic store contract

plugins/                       product plugins — namespaces unchanged
  transcription.asr/           AsrPort + stub/local/cloud adapters
                               + contributes stage.transcribe
  transcription.tts/           TtsPort + adapters + stage.synthesize
  llm.acp/                     EvalPort.judge + stage.judge
  storage.dataset/             DatasetStorePort impls (sqlite/R2/mem)
  patients/ consultations/ …    existing product plugins, untouched

plugins/eval.dom.consult/      eval-only domain pack
  schema.ts (Sample, gold entities), personas.ts, scriptgen.ts,
  templates/pt-BR/, locale.ts (LocaleProfile),
  metrics.ts (entity-match, negation — NOT WER),
  gate-policy.ts (clinical severity model)

evals/                         suite manifests — typed TS, data-shaped
  consult-asr.ts  consult-correction.ts  triage-extraction.ts …
  export default defineSuite({...}) — validated by eval-core Schema

apps/eval/                     @effect/cli composition root
  list | generate | run | gate | report | bench
```

## Manifest example (evals/consult-asr.ts)

```ts
defineSuite({
  id: "consult-asr",
  domain: "eval.dom.consult",
  dataset: { id: "consult-ptbr", version: "v1" },
  stages: ["synthesize", "transcribe", "correct", "judge"],
  caps: { asr: "transcription.asr", tts: "transcription.tts",
          judge: "llm.acp", store: "storage.dataset" },
  metrics: ["wer", "cer", "entity-f1", "negation"],
  gates: [ { id: "wer-floor", stage: "corrected", max: 0.05 },
           { id: "entity-precision", min: 0.90, minN: 40 },
           { id: "negation", policy: "consult.strict" } ],
})
```

## Key decisions

- **Stages contributed by owners**: `transcribe` ships with `transcription.asr`, `synthesize` with `transcription.tts`, `judge` with `llm.acp` — manifest only orders/selects.
- **Metrics owned by domain/capability**: WER/CER with transcription, entity/negation with consult domain — no `eval.metrics.*` namespace.
- **Capability id ≠ adapter**: `--profile stub|local|cloud` selects implementations globally; manifest overrides per-cap only when needed.
- **evals/ is CLI input**: `apps/eval` scans the dir, validates manifests via eval-core Schema, resolves plugin ids via the registry. Not registry citizens — no P-070 dependency for manifests themselves.
- **Severity model moves to domain pack**: gate engine stays policy-agnostic; `consult.strict` policy contributed by `eval.dom.consult`.
- **Dataset binding**: manifest references `{id, version}`; `eval generate` materializes into the store.

## Migration (incremental)

1. Extract `eval-core` (generic files; re-exports keep imports stable).
2. Move transcription domain → `eval.dom.consult`; ASR/TTS/LLM/store → product capability plugins (`transcription.*`, `llm.*`, `storage.dataset`); eval packages become consumers.
3. `evals/` manifests + `apps/eval` CLI absorbing `scripts/*.ts`.

## Acceptance

- Second suite runs with no core changes — often a manifest only.
- Eval exercises literally the product adapters (same plugin ids).
- Stage/metric/gate changes are manifest or plugin edits, never core edits.
- `apps/eval list|generate|run|gate|report|bench` works over `evals/*`.
- All existing eval tests stay green throughout.
