+++
id = "P-115"
title = "Transcript eval platform: hybrid synthetic/real dataset, hexagonal ports, staged clinical metrics"
status = "open"
created = "2026-11-13"
owner = "devin"
priority = "high"
labels = ["dataset", "eval"]
links = ["T-042"]
epic = "audio-pipeline"
+++

## Idea

A transcript-quality platform for the consultation pipeline: a hybrid
synthetic + real dataset, a hexagonal eval harness (ports & adapters),
and staged clinical-safety metrics — replacing the current "is it good?"
guesswork with a measurable, gated pipeline. Extends T-042 into a full
spec. Every non-deterministic task (script generation, judging) is
delegated through **Devin ACP sessions** — no isolated LLM SDK calls —
for controlled context delegation and traceable automation.

Target: final reviewed transcript ≥95% (exact metric definition is an
open decision — see §9), cheap to run, physician stays in the loop.

> NOTE: "JEV" is assumed to mean the structured judge/evaluation
> engine — modeled here as the `EvalPort`. Correct if it names a
> specific product/tool instead.

## Architecture — hexagonal (matches repo arch lint)

Domain lives in one package with zero infra imports; three adapter
packages at the edge. Mirrors the existing
`transcription`/`transcription-whisper` split.

```
packages/eval-dataset/            # domain only — no node/cf/process
  src/
    ports.ts        # TtsPort, AsrPort, LlmPort, EvalPort (JEV),
                    # DatasetStorePort, ReportPort — Effect service tags
    schema.ts       # manifest/sample/persona/run-record (effect/Schema)
    personas.ts     # persona matrix: region × age × register × specialty
    metrics.ts      # WER/CER, clinical-entity diff, calibration buckets
    diff.ts         # auditable reversible edit lists between stages
    pipeline.ts     # stage runners S0–S7 (pure orchestration via ports)
    acp.ts          # ACP task envelopes + run-record contract
packages/eval-dataset-local/      # Local adapters: fs/sqlite manifest,
                                  # whisper.cpp|faster-whisper HTTP, local TTS
packages/eval-dataset-cloud/      # Cloudflare adapters: Workers AI, R2,
                                  # Queues (reuses existing bindings)
packages/eval-dataset-stub/       # Stub adapters: deterministic, zero I/O
```

### Ports

| Port | Contract (sketch) | Local | Cloudflare | Stub |
|---|---|---|---|---|
| `TtsPort` | `synthesize(script, voiceProfile, acousticProfile) → AudioRef` | local TTS binary / HTTP TTS | external TTS endpoint or Workers AI audio model | canned 1s wav bytes |
| `AsrPort` | `transcribe(audioRef, {lang}) → {text, segments[{t0,t1,text,conf}]}` | whisper.cpp / faster-whisper server | Workers AI whisper-large-v3-turbo | canned transcript + conf 0.9 |
| `LlmPort` | `generate(envelope) → text`; `correct(raw) → EditList` | via ACP session (see §3) | via ACP session | deterministic canned output |
| `EvalPort` (JEV) | `judge({stage, output, reference}) → Verdict` (per-category) | via ACP session | via ACP session | canned verdict |
| `DatasetStorePort` | `put/get/list` manifest rows + blobs | filesystem + sqlite | R2 + D1 | in-memory map |
| `ReportPort` | `append(runRecord)` / `render(runId)` | markdown + JSONL on disk | R2 + PR comment | no-op |

## ACP execution model

- Every LLM-shaped task is an **ACP task envelope**:
  `{taskId, role: "generator"|"judge", inputs, outputSchema, seed,
  modelHint, costBudgetUsd}`.
- The orchestrator spawns Devin ACP sessions (`devin acp`, stdio
  JSON-RPC) per envelope; each completed session appends a `run-record`
  {inputHash, outputRef, tokens, estCostUsd, latencyMs, model} to the
  manifest — full provenance, no hidden API calls.
- `LlmPort`/`EvalPort` adapters submit envelopes; Stub returns fixtures
  so unit tests never spawn a session.

## Dataset schema (hybrid-ready: synthetic now, real later)

`samples` manifest row (SQLite/JSONL, versioned):

```
id, dataset_version
origin: "synthetic" | "real_anonymized"
script_text / script_hash           # ground truth
persona: {region, age_band, education_register, social_class_cues,
          gender, specialty}
acoustic_profile: {tts_model, voice_id, noise_profile
                   (clinic|phone_bandpass|far_mic), sample_rate, prosody}
audio_ref (nullable)
gold: {validated_by: "generator" | "physician:<id>", validated_at}
provenance (phase 2): {source_partner, anonymization_method,
                       lgpd_basis, consent_ref}
stages: {
  asr_raw:    {text_ref, engine, run_id},
  corrected:  {text_ref, engine, run_id, diff_ref},
  approved:   {text_ref, actor, run_id, diff_ref},   # = gold on synthetic
  anamnesis:  {doc_ref, engine, run_id}
}
```

Phase 2 ingests real anonymized consults via `DatasetStorePort`
(`origin="real_anonymized"`, `gold.validated_by="physician:*"`) — same
row, no schema change.

## Pipeline stages + side effects

| # | Stage | In → Out | Side effects |
|---|---|---|---|
| S0 | Script gen | persona → script | **ACP session** (LLM cost); manifest row write |
| S1 | TTS synth | script+acoustic → audio | TTS cost or local CPU; blob write (fs/R2) |
| S2 | Degradation | audio → audio' | local DSP (noise/resample); temp wav in `./tmp` (deleted post-run) |
| S3 | ASR | audio' → raw+conf | ASR cost (cloud) or CPU/RAM (local); run-record write |
| S4 | LLM cleanup | raw → EditList → corrected | ACP session cost; diff artifact write |
| S5 | Approval | corrected → approved | phase 1: auto = gold (no cost); phase 2: physician action (human time) |
| S6 | Anamnesis | approved → doc | ACP session cost |
| S7 | JEV eval | all stages + gold → verdicts+metrics | ACP session cost; report write; optional PR comment |

Run-environment side effects: `./tmp` scratch dir policy (create,
delete after run — nothing left on disk unless `--keep-tmp`); network =
ACP sessions + cloud adapters only; seeds pinned per run
(`seed`, `dataset_version`, git sha recorded in every run-record).

## Multi-stage eval — where each gain/loss lands

Per sample, JEV compares each stage to its reference:

1. **S3 vs script** — ASR baseline: WER/CER + clinical entities
   (negation, medication, dosage, frequency, unit, date →
   correct/substituted/dropped/invented).
2. **S4 vs S3** — cleanup audit: every edit classified
   `justified_fix | unnecessary | harmful` (hallucination);
   never-touch violations (number/dose/drug w/o flag) counted
   separately. Diffs are reversible — any edit can be rolled back and
   re-judged.
3. **S5 vs S4** — approval delta: what the auto pipeline still missed
   (phase 2; synthetic = zero delta by construction).
4. **S6 vs approved** — anamnesis field-level accuracy.

Critical clinical errors are counted **separately** from lexical
WER/CER — a transcript can be 98% WER-clean and still fatally wrong on
one dose.

## Confidence calibration (auto-accept)

- Bucket S3 segments by confidence (0.5–1.0, 0.05 steps); measure real
  accuracy per bucket against gold.
- Pick threshold maximizing **auto-accept precision** (target ≥99%
  correct-accept), not raw coverage — high-confidence false accepts are
  the worst clinical failure.
- Threshold is config + versioned; report = precision/coverage curve.

## Operational metrics per run

latency per stage · est. cost per consultation · failure/retry rate ·
chunk-boundary errors (segments split mid-sentence) · reproducibility
hash {git sha, dataset_version, seeds, model ids, prompt hashes}.

## CI/CD — opt-in gate

- Default `check` stays fast: stub adapters only — schema, diff engine,
  metrics math, determinism. Zero cost, zero network.
- Full eval suite opt-in: `workflow_dispatch`, PR label `eval:full`, or
  `KTB_EVAL=1`. Runs synth→ASR→JEV on a pinned subset; uploads report
  artifact + optional PR comment; fails only on configured regression
  deltas (WER, critical-error rate, precision@threshold).

## Phasing

- **Phase 1 (this spec):** package skeleton + stub + local adapters,
  script gen via ACP, ONE TTS engine (runtime accepts many), ~50–100
  samples, staged eval + diff audit, CI opt-in workflow.
- **Phase 2:** cloud adapters on by default, calibration tuned on real
  ASR confidence, real anonymized ingestion + physician approval loop.

## Open decisions

1. "≥95%" exact metric — recommend: WER ≤5% AND zero critical clinical
   errors per consult, reported per persona dimension.
2. JEV naming — confirm it = `EvalPort` judge as modeled here.
3. Single phase-1 TTS pick (Edge-TTS vs open TTS) — cost vs pt-BR voice
   variety.
4. Whether generation envelopes target `devin acp` sessions directly or
   a thin ACP client wrapper.

## Positioning cautions

- Documentation-only stance unchanged — eval measures transcription,
  never clinical correctness of content.
- Phase-2 real data needs anonymization + consent trail before
  ingestion (`provenance` fields exist precisely for that gate).

## Depends on

Extends T-042. Independent of P-016/P-017 implementation status —
consumes their artifacts (serverTranscript, correctedTranscript) as
eval subjects.
