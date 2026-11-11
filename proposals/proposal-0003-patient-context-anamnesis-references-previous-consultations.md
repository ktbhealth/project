+++
id = "P-003"
title = "Patient context: anamnesis references previous consultations"
status = "done"
created = "2026-11-08"
priority = "med"
+++

## Problem

Every consult is treated as an island: the anamnesis can't see
the patient's history. The physician retypes context that already
exists, and the product doesn't accumulate the asset that
differentiates it — continuous clinical memory per patient.

## Proposal

The anamnesis starts referencing the same patient's previous
consults:

- **Patient ↔ consults link**: the consult gets a stable
  `patient_id`; patients become their own entity (a `patients`
  plugin already exists — verify what's missing).
- **Context summary**: before/during structuring, the pipeline
  receives a summary of the last N consults (complaint, plan,
  cited medications) — not the entire record.
- **"Since last consult" sections**: the anamnesis highlights what
  changed (new symptoms, follow-up, adherence) instead of
  rewriting everything — the physician reviews the delta, not the
  volume.
- **Prompt discipline**: the LLM receives history as *factual
  context*, never as clinical suggestion. The documentation-only
  positioning holds: the system records and organizes, it doesn't
  decide.

## Scope (MVP)

1. `patient_id` on the consult + patient↔consults link
2. Endpoint/service aggregating a summary of the last 3 consults
3. Anamnesis template with a "history" block filled in
4. UI: "N previous consults" badge + collapsible context section
   in review

## Out of scope (for now)

- Full visual patient timeline
- Return/due-date alerts (edges into clinical decision)
- History sharing across physicians/clinics

## Acceptance criteria

- A returning patient's new consult shows a summary of previous
  consults on the review screen
- The generated anamnesis references history facts without
  inventing them
- With no linked patient, current behavior is unchanged
- Tests: history aggregation, block rendering, no-history fallback
