+++
id = "P-018"
title = "Transcript UI: original vs corrected tabs with processing status"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["audio-pipeline"]
epic = "audio-pipeline"
+++

## Idea

In the consultation transcript area (NewConsultationPage + ConsultationDetailPage), show two tabs: "Corrected" (default — LLM-cleaned text when available, else server transcript, else browser transcript) and "Original" (raw ASR output, always visible). Show async job status (processing/failed) while P-016/P-017 run.

## Likely scope

- Tabbed transcript viewer in plugins/consultations views; corrected tab default per product decision
- Status chip: "enhancing transcript…" / "enhancement failed — showing original"
- Generate-anamnesis uses whichever transcript the doctor is viewing (or explicitly the corrected one — decide UX)
- Audio playback control once P-015 lands (listen to source recording)

## Why it matters

Two texts without clear labeling confuses clinicians; tabs make provenance obvious — corrected is convenience, original is evidence. Defaulting to corrected reduces review friction without hiding the raw record.

## Positioning cautions

- Never label corrected text as "the transcript" unqualified — always show which version generated the anamnesis.
- If enhancement is still processing, the UI must not block dictation/save — degrade gracefully to original.
- i18n: all new strings through paraglide messages (en/pt-BR/es).

## Depends on

P-015 (playback), P-016 + P-017 (the corrected text + status it renders). Can ship incrementally: tabs with original-only first, corrected populates when ready.
