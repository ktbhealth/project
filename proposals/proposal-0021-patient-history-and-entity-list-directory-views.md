+++
id = "P-021"
title = "Patient history and entity list: directory views"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["directory"]
epic = "directory"
+++

## Idea

Two listing surfaces: (1) a patients directory — searchable/filterable
list of all registered patients with consult counts and last-visit
dates; (2) a per-patient history view — chronological consult list
with summaries (extends P-004's timeline into a navigable entity
list).

## Likely scope

- `/patients` index: table/cards, search by name, sort by last
  consult, filter by insurance/active
- History view: reuse ConsultationDetail summaries; click-through to
  full consult; optional date-range filter
- Server fns: `listPatients` gains search/sort params; `patientHistory`
  already exists for extraction — expose a UI-facing variant
- Empty states + i18n strings (paraglide, all 3 locales)

## Why it matters

Clinics think in panels ("my 200 patients"), not in consult IDs. The
directory is how a doctor finds the patient before the consult —
composes with P-010 day dashboard prep cards.

## Positioning cautions

- List endpoints stay owner-scoped; no cross-doctor visibility until
  P-022 registry exists.
- No batch export of the directory without explicit action — it's
  health data at scale.

## Depends on

P-019 (fields to display/filter), complements P-004/P-020.
