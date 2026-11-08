+++
id = "P-013"
title = "Visit summary: patient-facing shareable consult summary"
status = "open"
created = "2026-11-08"
priority = "low"
labels = ["platform-qol"]
+++

## Idea

A clean, patient-readable summary of the consult — what was
discussed, what was prescribed/plan — shareable as PDF or link.
The patient leaves with the record, not just the physician.

## Likely scope

- Template rendered from the approved anamnesis, PT-BR,
  layperson-readable (no raw clinical shorthand)
- Export PDF and/or signed link with expiry
- Optional: send via WhatsApp (ties into P-009 channel)

## Why it matters

Trust and retention: patients who receive a summary perceive a
modern clinic; physicians look organized. It's also the artifact
that travels — reinforces "the record is the business".

## Positioning cautions

- Summary renders ONLY from physician-approved content —
  nothing auto-generated from raw transcript.
- Patient-facing text needs a review step: clinical terms must
  be translated to plain language by the physician or a
  deterministic glossary, never an LLM rewriting freely.

## Depends on

Approved-anamnesis export pipeline; optional P-009 for delivery.
