+++
id = "ADR-0001"
title = "Transcription sources as a port"
status = "done"
created = "2026-11-06"
priority = "medium"
+++

# ADR-0001: Transcription sources as a port

## Decision

Every audio source converts into the same stream of
`TranscriptEvent`s and enters the same pipeline
(`AnamnesisExtractor` → physician review). Adapters:

- `WebSpeechTranscriber` (MVP): browser, in-person, client-side.
- `MeetingBotTranscriber` (future): a bot joins the
  Meet/Zoom/Teams call via meeting-bot-as-a-service (Recall.ai
  et al.), transcript arrives by server-side webhook.
- `UploadTranscriber` (future): recording upload → Whisper.

## Rationale

Teleconsult is the natural use case (the consult is already
digital), but building our own bot (Puppeteer + audio capture) is
week(s) of fragile work. Bot-as-a-service costs ~$0.70–1/h per
call — becomes a pricing line (in-person free, bot per consult).
LGPD bonus: the bot visible in the call IS the consent.

## Consequences

Client and server `Transcriber`s converge conceptually — what
matters is the event stream entering the pipeline, not the side
where the adapter runs. `consulta/nova` (live browser) is a
special case of "source". Consent is by-design: bot source =
visible; browser/upload source = physician declares consent.
