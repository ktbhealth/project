+++
id = "P-001"
title = "Own call platform: be the channel, not the capturer"
status = "open"
created = "2026-11-06"
priority = "high"
labels = ["business", "strategy"]
+++

# P-001: Own call platform — be the channel, not the capturer

## Idea

Instead of capturing other people's calls (fragile matrix: tab
audio, WASM, bot, loopback), the product becomes the teleconsult
room:

- the physician sends a link via WhatsApp (that's already the routine)
- the patient joins our call via browser (zero install)
- the physician is in our app
- natively separated audio legs = perfect labels,
  zero diarization, consent becomes a banner, clean LGPD

## Trade-offs

**Pros**: eliminates the ENTIRE capture matrix; controlled
quality; covers WhatsApp/phone/browsers without exception; moat —
the call AND the documentation live in the same place.

**Cons**: stops being scribe-alongside → becomes a teleconsult
platform (scheduling, waiting room, reliable video); competes
with Doctoralia-video/Conexa on the channel.

## Path

LiveKit/Daily/Twilio for infra (weeks, not months). Strategy:
scribe-alongside as the wedge → if the pilot shows physicians are
willing to send OUR link instead of the Meet link, the platform
wins. Validation question in interviews: "would you send our link?"

## Verdict (2026-11-06)

Endgame, not wedge. Risk: video calls are a merciless problem —
if it drops, the blame is ours. Mitigation: **audio-only first**
(voice without video — much anamnesis doesn't need video, 10x more
reliable/cheaper to operate). Gate: pilot confirms physicians send
our link; if not, scribe-alongside is the ceiling.

## Phases

1. validate via pilot (the link question)
2. audio-only 1:1 call (LiveKit/vonage voice) + scribe per leg
3. optional video + scheduling + waiting room + medical record

## Positioning (2026-11-06)

Don't innovate on the call — the call is commodity by design,
generic-and-reliable is the goal. The innovation is what the call
PRODUCES: someone else's call ends in documentary silence; ours
ends with the reviewed anamnesis. Differentiators: async
pre-consult (chat triage → pre-filled anamnesis), assist during
the call, longitudinal per-patient context (compounding moat), and
transcription+evidence as a legal record. Framing:
"generic telemedicine on the outside, smart documentation on the
inside" — the call is the hook, the record is the business.
