+++
id = "P-004"
title = "Patient timeline UI: full consultation history view"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["v2"]
+++

## Idea

Full chronological patient view: all consults as a visual
timeline (date, complaint, plan, documents), not just a summary
of the last 3 visits.

## Likely scope

- Per-patient list/page with all consults ordered chronologically
- Each entry: date, summarized anamnesis, link to the consult
- Filters by period/type if it makes sense

## Depends on

Stable patient model (patient_id + history aggregation).

## Cautions

- Volume: patients with many consults need pagination/aggregation
- Positioning: it's record visualization, not decision —
  no "risk" or "overdue" badges
