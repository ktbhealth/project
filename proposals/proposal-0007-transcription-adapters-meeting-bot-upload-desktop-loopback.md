+++
id = "P-007"
title = "Transcription adapters: meeting bot, upload, desktop loopback"
status = "open"
created = "2026-11-08"
priority = "med"
+++

## Idea

Expand transcription sources beyond the browser (see ADR-0001,
which defines the `TranscriptEvent` port → single pipeline):

- `MeetingBotTranscriber` — a bot joins Meet/Zoom/Teams calls via
  meeting-bot-as-a-service (Recall.ai et al.); the transcript
  arrives by server-side webhook. ~$0.70–1/h per call → pricing
  line; visible bot = built-in consent (LGPD).
- `UploadTranscriber` — recording upload → server-side Whisper.
- `SystemAudioTranscriber` — desktop loopback (Tauri, see T-013
  in the backlog) capturing any call app.

## Notes

- ADR-0001 already fixed the contract: adapters emit the same
  stream; pipeline/anamnesis/review don't change.
- The bot is the answer for calls outside the physician's browser.
- Upload is the universal fallback and the base for demos/recordings.

## Depends on

Nothing blocks starting with upload (simplest); bot and desktop
are bigger.
