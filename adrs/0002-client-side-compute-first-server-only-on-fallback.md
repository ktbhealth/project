+++
id = "ADR-0002"
title = "Client-side compute first, server only on fallback"
status = "done"
created = "2026-11-06"
priority = "med"
+++

# ADR-0002: Client-side compute first, server only on fallback

## Decision

STT runs on the client whenever possible; the server only
processes audio when the cheap side's quality fails. Separate
sources give speaker labels without diarization:

- `WebSpeechTranscriber` — physician's mic (free, native, pt-BR)
- `TabAudioTranscriber` — teleconsult tab audio via
  Whisper WASM (transformers.js) → "Patient" side
- `MediaRecorder` — local recording; uploads to the server ONLY
  if confidence is low/gaps → Whisper/Deepgram re-processes the
  excerpt

## Rationale

Cost: the happy path is ~free (client compute), paid server only
on the bad cases — undercuts competitors that always charge for
STT. LGPD: patient audio never leaves the device in the common
case; it only uploads on demand ("when necessary"), which remains
sellable.

## Constraints

- Web Speech API is mic-only — it doesn't accept MediaStream; tab
  audio requires Whisper WASM (~40-75MB, heavy CPU). Accepted
  trade-off.
- "Both sides install it" was rejected: the patient installs
  nothing — only the physician runs the app.
- Fallback assumes local recording is retained until confidence
  is evaluated — discard after approval (data minimization).

## Consequences

The `Transcriber` port gains multiple per-source instances; merge
by speaker label at pipeline input. Meeting bot (ADR-0001) remains
an option for calls outside the physician's browser or without
tab audio.
