+++
id = "P-063"
title = "Doctor-side whisper-large-v3-turbo live transcription with server fallback"
status = "open"
created = "2026-11-12"
owner = "devin"
priority = "med"
labels = ["product"]
links = ["P-007", "P-016", "P-017"]
+++

## Context

Live in-consult transcription today runs `onnx-community/whisper-base` via
transformers.js in the doctor's browser (`@ktb/transcription-whisper`),
with Web Speech API as last resort. Accuracy on pt-BR medical vocabulary
is limited. The authoritative record is already server-side:
recording → R2 → queue → `@cf/openai/whisper-large-v3-turbo` → LLM
cleanup (`serverTranscript` / `correctedTranscript`, P-016/P-017).

## Decision

- Upgrade the doctor-side client model to
  `onnx-community/whisper-large-v3-turbo` (~1.6GB fp32; ship q8/q4
  dtype to cut download). WebGPU → WASM fallback already exists in
  `whisper-worker.ts`.
- Compute runs **only on the doctor's machine** — the patient never runs
  transcription; the transcript is shared with the patient afterwards.
- Audio is **always recorded and stored**: the existing upload → R2 →
  queue → server turbo + LLM cleanup pipeline stays untouched and
  remains the authoritative record.
- Fallback chain: local turbo → local whisper-base → webspeech; server
  transcription remains the fallback for low-quality sessions
  (ADR-0002) and the record of truth.

## Scope

- Parametrize model + dtype in `packages/transcription-whisper`
  (deploy/host config, no provider literals in ports).
- Device capability gating (`navigator.gpu` + memory heuristic) with
  automatic degrade to whisper-base.
- Prefetch/warm-up strategy for the ~1GB first download post-login
  (extend `whisper-prefetch.ts`).
- Clinic broadband assumed; no patient-side compute, no new vendors.

## Non-goals

- Patient-side transcription compute.
- Speaker diarization (labels still come from stream origin: mic =
  doctor, tab = patient).
- Streaming/server-side live STT (rejected: latency + cost; server
  stays batch/authoritative).
