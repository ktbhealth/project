+++
id = "P-007"
title = "Transcription adapters: meeting bot, upload, desktop loopback"
status = "open"
created = "2026-11-08"
priority = "medium"
+++

## Ideia

Expandir as fontes de transcrição além do browser (ver ADR-0001,
que define o port `TranscriptEvent` → pipeline única):

- `MeetingBotTranscriber` — bot entra na call Meet/Zoom/Teams via
  meeting-bot-as-a-service (Recall.ai et al.); transcript chega
  por webhook server-side. ~$0.70–1/h por chamada → linha de
  pricing; bot visível = consentimento embutido (LGPD).
- `UploadTranscriber` — upload de gravação → Whisper server-side.
- `SystemAudioTranscriber` — loopback desktop (Tauri, ver T-013
  no backlog) capturando qualquer app de call.

## Notas

- ADR-0001 já fixou o contrato: adapters emitem o mesmo stream;
  pipeline/anamnesis/review não mudam.
- Bot é a resposta para calls fora do browser do médico.
- Upload é o fallback universal e base para demos/gravações.

## Depende de

Nada bloqueia começar pelo upload (mais simples); bot e desktop
são maiores.
