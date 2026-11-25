+++
id = "PL-019"
title = "Transcript eval platform: M0 stubs-first, M1 synth dataset, M2 Jev eval, M3 calibration+CI, M4 cloud+real ingest"
status = "open"
created = "2026-11-13"
owner = "devin"
priority = "high"
links = ["T-042", "P-066"]
+++

## Execution plan — P-066 transcript eval platform

Ordered so every step is testable against **stub adapters first**
(zero cost, zero I/O, low RAM — dev machine constraint). Live
adapters land only after the domain is proven. Living spec:
`transcription-eval-platform-spec.md` (workspace root).

### M0 — Domain + stubs (foundation, no infra)

- [ ] `packages/eval-dataset`: effect/Schema models — manifest row,
      persona, acoustic_profile, stage artifacts, run-record, verdicts
- [ ] Port tags (Effect services): `TtsPort`, `AsrPort`, `LlmPort`,
      `EvalPort` (Jev), `DatasetStorePort`, `ReportPort`
- [ ] `diff.ts`: reversible edit-list engine (apply/rollback spans)
- [ ] `metrics.ts`: WER/CER, clinical-entity diff classification,
      confidence bucketing
- [ ] `pipeline.ts`: stage runners S0–S7 as pure port orchestration
- [ ] `packages/eval-dataset-stub`: deterministic adapters — canned
      wav, canned transcript+conf, canned EditList, canned Jev
      answers, in-memory store, no-op report
- Verify: `cd packages/eval-dataset && bunx vitest run` — S0–S7 run
  end-to-end on stubs

### M1 — Synthetic dataset generation

- [ ] `personas.ts`: matrix — region × age_band × register × specialty
- [ ] `LlmPort` ACP adapter: envelope → `devin acp` session → script
      text + run-record (input hash, cost, latency, seed)
- [ ] Script-gen runner → manifest rows (`origin=synthetic`,
      `gold.validated_by="generator"`)
- [ ] `TtsPort` local adapter: ONE engine (pick: Edge-TTS vs open TTS —
      open decision #3 in spec) + ffmpeg degradation (clinic noise,
      phone bandpass, resample)
- [ ] `DatasetStorePort` local: filesystem blobs + SQLite manifest
- Verify: `bun run eval:generate -- --count 10 --stub-llm` produces
  manifest + audio; `--acp` variant logs run-records

### M2 — Eval engine

- [ ] `AsrPort` local adapter: faster-whisper/whisper.cpp HTTP
      (OpenAI-compatible `/v1/audio/transcriptions`, word timestamps)
      — same contract `createLocalAi` already proves
- [ ] `EvalPort` Jev adapter: `/v1/systemone` client, question
      builders per stage (`Noul` per entity, `Choice` per edit,
      `Score` severity), answers → verdicts + run-records
- [ ] Multi-stage eval runner: S3-vs-script / S4-vs-S3 diff audit /
      S5 delta / S6 field accuracy; critical errors counted separately
- [ ] `ReportPort` local: markdown + JSONL per persona dimension
- Verify: 10-sample run produces per-stage report; stub suite still
  green

### M3 — Calibration + CI gate

- [ ] Calibration report: accuracy per confidence bucket →
      precision-first threshold recommendation (config + versioned)
- [ ] CI: stub suite inside default `check`; opt-in eval workflow
      (`workflow_dispatch`, PR label `eval:full`, `KTB_EVAL=1`) on a
      pinned subset; artifact upload + regression deltas
- [ ] Repro manifest per run: git sha, dataset_version, seeds, model
      ids, prompt hashes
- Verify: PR with `eval:full` label runs the suite and posts report

### M4 — Cloud adapters + phase-2 prep

- [ ] `packages/eval-dataset-cloud`: Workers AI ASR, R2 store,
      report → R2 (+ optional PR comment)
- [ ] Real-ingestion path: `origin=real_anonymized` importer with
      provenance gate (anonymization_method, consent_ref required)
- [ ] S5 physician-approval hook (real actor replaces auto-gold)
- Verify: same suite runs against cloud adapters via env switch

### Dependencies / parallelization

- M1 personas+schema (M0) blocks everything; ACP adapter ∥ TTS adapter
- M2 needs M1 samples; Jev adapter ∥ ASR adapter
- M3 needs M2 reports; M4 independent of M3, blocked only by M0

### Candidate todos (split on approval)

scaffold domain pkg · stub adapters · personas+ACP script-gen ·
TTS local + degradation · local store · ASR local adapter ·
Jev adapter + question builders · eval runner + report ·
calibration report · CI opt-in workflow · cloud adapters ·
real-ingestion gate

### Open decisions to resolve before M1

1. "≥95%" exact metric (proposal: WER ≤5% AND zero critical clinical
   error per consult, per persona dimension)
2. `EvalPort` exempt from ACP delegation (recommended — Jev is a
   decision API, not generative)
3. Phase-1 TTS pick (Edge-TTS vs open TTS)
4. ACP envelopes direct vs thin client wrapper

### Verification

- Every M has a `bunx vitest run` gate scoped to its package — repo
  coverage gate is 100% repo-wide, never chase it globally.
- Each adapter lands with a stub-parity test: same inputs → same
  output *shape* as the stub (schema-level), so CI never needs the
  live backend.
