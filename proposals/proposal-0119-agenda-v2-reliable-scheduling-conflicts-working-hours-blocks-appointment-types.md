+++
id = "P-119"
title = "Agenda v2: reliable scheduling — conflicts, working hours, blocks, appointment types"
status = "open"
created = "2026-11-26"
owner = "human:vini"
priority = "med"
labels = ["platform-qol", "product"]
links = ["T-031", "P-008", "P-009", "P-010"]
epic = "agenda"
+++

## Idea

P-008 shipped the agenda base (plugins/agenda). Make it trustworthy for daily use before adding channels/sync.

## Scope (Phase 1)

- A: overlap/conflict rejection in Appointments.create/update (live statuses only), surfaced in UI
- B: doctor working hours (weekday, start, end) + agenda shading + free-slot suggestion
- C: time blocks (lunch/surgery/vacation) without a patient
- D: appointment types with default duration

## Later phases

- Phase 2: Today view (P-010)
- Phase 3: patient portal appointments + patient reminders (P-009)
- Phase 4: Google Calendar adapter on @ktb/calendar (P-008 remainder)

## Constraints

Hackathon freeze Nov-30 → Dec-12: work on a branch, merge after freeze. Scheduling is logistics, no triage semantics.
