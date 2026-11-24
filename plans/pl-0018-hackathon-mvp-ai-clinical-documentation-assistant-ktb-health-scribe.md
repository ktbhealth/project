+++
id = "PL-018"
title = "Hackathon MVP — AI clinical documentation assistant (KTB Health scribe)"
status = "open"
created = "2026-11-24"
owner = "devin"
priority = "high"
labels = ["hackathon", "mvp"]
links = ["T-092", "T-193", "T-196", "T-197", "P-065", "P-114"]
epic = "mvp"
milestone = "hackathon"
due = "2026-11-30"
+++

## Context

Hackathon brief: AI Medical Scribe — "let doctors focus on patients, while AI handles clinical documentation." KTB Health already implements ~80% of the recommended MVP; this plan maps brief → product reality, fixes the cut line, and sequences delivery under milestone `hackathon` / epic `mvp`.

## Brief → product mapping

| Brief | KTB Health reality |
|---|---|
| Record w/ consent | patient_access invites + `consentAt` recording consent; ZK vault for tenant encryption |
| Speech-to-text | Client whisper (webgpu/wasm) + tab-loopback capture, Doctor:/Patient: speaker labels per stream; server whisper-large-v3-turbo fallback + queued pipeline; PT-BR + clinical glossary `initial_prompt` (PR #287) |
| Speaker identification | Source-stream diarization (ADR-0002): mic = doctor, tab/bot = patient |
| SOAP note | 8-section anamnesis (queixa_principal, hda, antecedentes, alergias, medicamentos_em_uso, exame_fisico, hipoteses, conduta) — SOAP-equivalent, locale-native; every section carries `evidence` = verbatim transcript citations ("never invents findings" is structural, not a prompt hope) |
| Review/approve | Inline edit pre-save + LLM cleanup pass (correctedTranscript vs original tabs). Batch approve queue = P-014 → CUT |
| Export | PDF export + **FHIR R4 Bundle** (`packages/interop-fhir`) + openEHR composition — stronger than "clipboard export" |
| Brazilian differentiation | pt language, CID-10/TUSS glossary, SUS/convênio vocabulary, LGPD-shaped consent+encryption story |
| Specialty templates | P-114 demo via T-042 specialty dataset — NOT T-091 pack architecture (post-hackathon) |

## Stack correction (for pitch accuracy)

Brief suggests Python/FastAPI/Postgres/OpenAI — actual stack is stronger for the story: TanStack Start on Cloudflare Workers, Workers AI whisper, Effect-TS services behind ports, D1+R2, bun monorepo, 100% coverage gate, mutation-tested packages.

## Cut line (deferred, handoffs recorded)

P-005, P-006, P-011, P-012, P-014, P-024 — all post-MVP. T-013 desktop app post-pilot.

## Phases

- [ ] Demo reliability: T-193 (all sources live), T-092 (seeded env), T-095 (runbook), T-196 (backup video)
- [ ] Differentiator: P-114 (specialty demo), T-164 (real WER), PR #287 glossary
- [ ] Pitch/traction: T-197 (judging criteria), T-194 (triage), T-041/T-089 (interviews/pilot), T-192 (wedge copy), presentation-draft + website-draft
- [ ] Metrics: time-to-document vs manual baseline on synthetic consult (see tasks)
