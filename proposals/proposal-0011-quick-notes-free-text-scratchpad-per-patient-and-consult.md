+++
id = "P-011"
title = "Quick notes: free-text scratchpad per patient and consult"
status = "open"
created = "2026-11-08"
priority = "low"
labels = ["platform-qol"]
+++

## Idea

A lightweight scratchpad for observations that don't fit the
structured anamnesis: admin notes, patient preferences, hallway
context ("prefers female staff", "daughter translates").

## Likely scope

- Note entity scoped to patient or to a specific consult
- Rich-enough text (bold/lists), timestamps, author = physician
- Surfaced in prep card (P-010) and consult review
- Distinct from anamnesis: never enters the clinical record or
  the exported document unless explicitly promoted

## Why it matters

Physicians keep this context in their heads or on paper today.
Capturing it in-product compounds the longitudinal context moat
(P-003/P-004) without touching clinical structure.

## Positioning cautions

- Notes are explicitly non-clinical: label them so; no fields
  that mimic vitals or symptoms.
- Review/export must make clear what is record vs. scratch.

## Depends on

Patient model; optional consult link (works patient-scoped
without P-008).
