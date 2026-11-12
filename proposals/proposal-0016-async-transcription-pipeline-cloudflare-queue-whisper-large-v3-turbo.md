+++
id = "P-016"
title = "Async transcription pipeline: Cloudflare Queue + Whisper large-v3-turbo"
status = "done"
created = "2026-11-08"
priority = "high"
labels = ["audio-pipeline"]
epic = "audio-pipeline"
+++

## Idea

After audio lands in R2 (P-015), enqueue a job on a Cloudflare Queue; a consumer runs Workers AI Whisper large-v3-turbo on the recording and stores the result as the "server transcript". This is the "bigger model" pass — higher accuracy than on-device whisper-base, especially for phone audio and accented PT-BR.

## Likely scope

- Queue + consumer binding in alchemy.run.ts; job = { consultationId, audioKey }
- Consumer: fetch audio from R2 → ai.run("@cf/openai/whisper-large-v3-turbo") → persist transcript + job status (D1: pending/processing/done/failed)
- Status surfaced to UI (P-018 polls or receives update)
- Idempotent consumer (retries safe), dead-letter handling, size guard (skip/flag oversized recordings)

## Why it matters

Browser whisper-base is the fast/cheap path; large-v3-turbo is the accuracy path. Async queue decouples a 30–90s server transcription from the consult flow — doctor keeps working while it runs.

## Positioning cautions

- Server transcription sends audio to Cloudflare — disclose in privacy copy; it already happens today via /api/transcribe fallback, this just makes it systematic.
- Job status must be honest: "processing" vs "failed" vs "done" — never show a stale/partial transcript as final.
- Cost control: queue only fires when audio was actually stored (P-015 consent path), not for every keystroke.

## Depends on

P-015 (audio must exist in R2). Blocks P-017 (LLM cleanup consumes its output) and P-018.
