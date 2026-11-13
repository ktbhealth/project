+++
id = "P-008"
title = "Doctor agenda: scheduling and appointment management"
status = "open"
created = "2026-11-08"
priority = "med"
labels = ["platform-qol", "product"]
+++

## Idea

A doctor-facing agenda inside KTB Health — but built as a
**calendar port with adapters**, not a walled garden. KTB's own
appointment entity stays the source of truth for consult links and
statuses; external calendars sync through adapters, starting with
**Google Calendar** (what most Brazilian clinicians actually use).

## Likely scope

- `CalendarPort` interface (list/create/update/cancel events,
  watch/webhook or poll for external changes) — same port-adapter
  pattern as ADR-0001 transcription sources
- `GoogleCalendarAdapter` first: OAuth connect, two-way sync of
  appointment ↔ Google event (title = patient initials/type, no
  clinical data in event descriptions — LGPD)
- Later adapters: Outlook/Graph, CalDAV/Apple, iCloud — same port
- Week/day calendar view with consult blocks; appointment entity:
  datetime, type, patient link, status
  (scheduled / confirmed / completed / no-show)
- Conflicts shown read-only; KTB never silently overwrites an
  external event

## Why it matters

Agenda is the spine of the daily loop: prep view, reminders, and
the consult itself all hang off it. Syncing with Google Calendar
means the doctor keeps their existing habit — KTB layers consult
context on top instead of demanding migration. Port-adapter keeps
us uncommitted to any single calendar vendor.

## Positioning cautions

- Scheduling is logistics, not triage: no waitlist prioritization
  or "urgency" scoring — that edges into clinical decision.
- No-show tracking is administrative fact, not a patient judgment.
- Calendar event payloads must not leak PHI: patient name only
  where the doctor's own calendar already shows it; prefer initials
  + internal ID link.
- OAuth tokens are credentials — store encrypted, minimal scopes
  (calendar.events only).

## Depends on

Patient model (P-003 done) for patient links; Google OAuth app
setup (external dependency). Later adapters are independent
proposals once the port proves out.
