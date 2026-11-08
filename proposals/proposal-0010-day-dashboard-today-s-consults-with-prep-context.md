+++
id = "P-010"
title = "Day dashboard: today's consults with prep context"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["platform-qol"]
+++

## Idea

A "today" view that assembles the physician's day in one screen:
agenda order, per-patient prep card (last consult summary, open
items), and one-click start for recording/documentation.

## Likely scope

- Ordered list of today's appointments with status
- Prep card per patient: last anamnesis summary, medications
  cited, pending return date (from P-005 when it lands)
- "Start consult" action → opens consult flow with patient
  pre-linked
- Empty states: no agenda → prompt to schedule; no patients →
  generic consult

## Why it matters

This is where agenda + patient context (P-003) + documentation
meet: the physician opens KTB in the morning and the whole day
is already staged. It's the retention surface — daily habit,
not weekly chore.

## Positioning cautions

- Prep cards surface recorded facts only; no generated
  "suggested questions" or risk flags.
- Ordering is chronological/manual — never auto-prioritized by
  inferred severity.

## Depends on

P-008 (agenda), P-003 (patient context — done), consult flow.
