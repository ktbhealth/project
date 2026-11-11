+++
id = "P-009"
title = "Patient appointment reminders before consults"
status = "open"
created = "2026-11-08"
priority = "med"
labels = ["platform-qol"]
+++

## Idea

Automated reminders to the patient before a scheduled consult —
the channel the patient already lives in (WhatsApp first, email
fallback) — with confirm/reschedule affordances. Triggered off the
agenda (P-008), which syncs with Google Calendar and other providers
through the calendar port — reminders follow the appointment
wherever it was created.

## Likely scope

- Reminder rules: T-24h / T-1h configurable per appointment type
- Message templates in PT-BR; include consult link when tele
- Patient confirm/cancel action → updates appointment status
  (and the synced calendar event via the port)
- Opt-out honored (LGPD: patient controls notifications)
- Delivery via a notification port too: WhatsApp adapter first,
  email/SMS adapters later — same port-adapter convention

## Why it matters

No-shows are pure lost revenue for the physician; reminders are
the highest-ROI QoL feature and naturally extend the agenda
(P-008). The consult link in the reminder is also the natural
on-ramp to the platform channel (P-001). Because the agenda syncs
with Google Calendar, reminders also fire for appointments the
doctor created outside KTB.

## Positioning cautions

- Reminders are logistics (time/place/link), never clinical
  content — no prep questions that imply diagnosis.
- Sending messages on the physician's behalf needs explicit
  consent scope and an audit trail.
- WhatsApp Business API costs per conversation — track as COGS;
  email fallback keeps the feature viable at zero marginal cost.

## Depends on

P-008 (agenda/appointments + calendar port), P-019 (patient
register — needs a reachable contact channel), a notification
channel port (WhatsApp Business API adapter first, email second).
