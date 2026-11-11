+++
id = "P-017"
title = "LLM transcript cleanup: corrected text alongside original"
status = "open"
created = "2026-11-08"
priority = "med"
labels = ["audio-pipeline"]
epic = "audio-pipeline"
+++

## Idea

Run the configured LLM (Anthropic/OpenAI env keys already wired for anamnesis) over the server transcript to produce a corrected version: fixes ASR errors, punctuation, speaker-flow artifacts — output stored as a separate field, never overwriting the original.

## Likely scope

- Cleanup step chained after P-016 consumer (same queue message or second stage)
- Prompt contract: PT-BR medical conversation, preserve clinical terms verbatim, no summarization, no content invention — correction only
- Storage: consultations.transcript_corrected (or JSON sidecar) + which engine produced it
- Fallback: if LLM unavailable, corrected = null and UI shows original only

## Why it matters

ASR output is raw evidence; the corrected text is what the physician actually reviews when generating the anamnesis. Keeping both preserves auditability (original = what was said) while giving a readable default.

## Positioning cautions

- "Corrected" must be labeled as machine-cleaned — it is NOT physician-approved and must not feed anamnesis generation silently; the doctor chooses which text to use.
- LLM cleanup must not add/remove clinical content — constrain the prompt to surface correction; diff view optional later.
- Cost/latency: run only on the server transcript (one pass per consult), not per chunk.

## Depends on

P-016 (needs the large-model transcript as input; can also accept browser transcript as input if server ASR fails — decide in implementation). Blocks P-018.
