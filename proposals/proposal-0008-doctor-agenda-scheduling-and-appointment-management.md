+++
id = "P-008"
title = "Doctor agenda: scheduling and appointment management"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["platform-qol"]
+++

## Idea

A doctor-facing agenda inside KTB Health: the physician manages
their own consult schedule — slots, types (in-person / tele / phone),
patient assignment — without depending on an external booking tool.

## Likely scope

- Week/day calendar view with consult blocks
- Appointment entity: datetime, type, patient link, status
  (scheduled / confirmed / completed / no-show)
- Manual entry first; recurring schedules if needed
- Link appointment ↔ consult record ↔ patient

## Why it matters

Agenda is the spine of the daily loop: prep view, reminders, and
the consult itself all hang off it. Today the doctor keeps the
schedule in WhatsApp/phone/calendar — bringing it in makes KTB
the workspace, not just the scribe.

## Positioning cautions

- Scheduling is logistics, not triage: no waitlist prioritization
  or "urgency" scoring — that edges into clinical decision.
- No-show tracking is administrative fact, not a patient judgment.

## Depends on

Patient model (P-003 done) for patient links; nothing else blocked.
