# Business Plan — Anamnesis AI

*Status: draft — pending validation of the open decisions at the end.*

## Problem

Brazilian doctors spend ~40% of the consult documenting. Poorly
recorded anamnesis = clinical error, legal exposure, and a 15-min
consult turning into 25 min of charting. Brazilian EMRs have bad
UX — the physician types looking at the screen, not at the patient.

## Solution

Ambient scribe: the app records the consult → transcribes (PT-BR)
→ structures the clinical anamnesis (CC, HPI, background, physical
exam, hypotheses, plan) → physician reviews and approves →
exports to the medical record/EMR.

## Who pays

- **MVP**: individual physician (B2C) — self-serve, Freed-style
  model (~R$99–149/month, product-led, low CAC)
- **Target**: small/medium clinics (B2B) — per-seat subscription

## Competition

- **Global**: Abridge (~$5B valuation), Nuance DAX (Microsoft),
  Nabla, Heidi, Ambience, Freed, Suki — a war with hundreds of
  millions in funding.
- **Brazil**: less mature, but EMRs (Feegow/Doctoralia, iClinic,
  Amplimed, MV, Tasy) are racing to embed scribe — risk of it
  becoming a medical-record feature.
- **Commoditization**: Whisper + LLM do 80% of this in a weekend —
  the product can't be just "transcribes".

## Differentiation (pick 1–2, not all)

1. **Deep clinical PT-BR** — Brazilian anamnesis taxonomy, CID-10,
   local semantics; a proprietary corpus becomes the moat.
2. **Specialty-first** — one specialty with a rich template
   (psychiatry, derm, cardio…), where the anamnesis structure is
   the value, not the transcription.
3. **LGPD-native** — patient consent, anonymization/edge, audit
   trail — an institutional selling point.
4. **EMR integration** — write into existing medical records
   (Feegow, iClinic…) instead of competing with them.

## Model

- Freemium: N free consults/month → R$99–299/month per physician.
- B2B: per seat/month + onboarding; per-consult at high volume.
- Margin: transcription (~$0.006/min) + LLM ≈ cents per consult →
  normal SaaS margin.

## Main risks

- **Regulatory**: LGPD — sensitive health data (consent, DPO,
  minimization). CFM does not require certification for auxiliary
  documentation today, but a model error = liability that splashes
  onto us through the physician.
- **Platform competition**: EMR embeds scribe for free → we need
  to be better or integrate.
- **Clinical accuracy**: hallucination in an anamnesis isn't a
  bug, it's an incident — every field must point to the audio
  excerpt that grounds it (source-linking).

## MVP (hypothesis)

Audio upload/recording → PT-BR transcription → editable structured
anamnesis (1 specialty) → export/copy → metrics: % of fields
accepted without editing, self-reported time saved, D7 retention
of paying physicians.

## Kill / validate

10 physicians using it for 2 weeks: if nobody pays, or editing
time > typing time, it dies.

## Decisions (2026-11-06)

- **Model**: Freed — self-serve, individual physician, product-led.
- **Positioning**: documentation tool, NEVER decision support —
  every anamnesis is a draft reviewed/approved by the physician.
  Outside SaMD/ANVISA; human-in-the-loop is the liability shield.
- **Telemedicine-first**: teleconsult is the easy mode — per-participant
  audio channels = free speaker labels (diarization solved) +
  visible bot = built-in consent.
- **In-person = dictation mode**: physician narrates alone
  post-consult (single speaker — Web Speech is enough). Diarization
  on a single mic is a known problem and NOT a priority.
- **Moat**: not tech (it's all commodity) — it's the channel +
  accumulated patient context across consults + embedding in the
  workflow.

## Validation signal (2026-11-06)

Informal interviews: physicians say "relevant", ALREADY TRIED
competitors (likely foreign, USD) and found them expensive.
Reading: real pain + market awareness + price sensitivity = opening
for a local undercut (R$99–149 vs. ~R$500+ for the foreign ones).
Still not proof of purchase — validate via pilot: 2 weeks free with
the same physicians, asking (1) which competitor they tried,
(2) whether price+language or only price was the problem,
(3) how much they would pay.

## Channel research (2026-11-06)

BR reality: WhatsApp dominates remote care (~69% of physicians —
APM) and is UNCAPTURABLE (no bot, no tab). Browser-based calls
(Meet/Zoom/Doctoralia-video) are the capturable minority via tab
audio/bot. Consequence: post-consult dictation + recording upload
become the primary path; tab audio/bot cover the browser segment.
The decisive routine integration is "copy to the EMR" (clipboard),
not API. Key pilot question: "which app does the consult run on?"

## Open decisions

1. **MVP specialty** — whichever we get access to real physicians in.
2. **LGPD as a feature** — auditable/on-prem from day 1, or
   minimal compliance in the MVP?
3. **Transcription stack** — teleconsult bot (Recall.ai) vs.
   upload+Whisper vs. Deepgram with diarization.
