+++
id = "P-009"
title = "Patient appointment reminders before consults"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["platform-qol"]
+++

## Idea

Automated reminders to the patient before a scheduled consult —
the channel the patient already lives in (WhatsApp first, email
fallback) — with confirm/reschedule affordances.

## Likely scope

- Reminder rules: T-24h / T-1h configurable per appointment type
- Message templates in PT-BR; include consult link when tele
- Patient confirm/cancel action → updates appointment status
- Opt-out honored (LGPD: patient controls notifications)

## Why it matters

No-shows are pure lost revenue for the physician; reminders are
the highest-ROI QoL feature and naturally extend the agenda
(P-008). The consult link in the reminder is also the natural
on-ramp to the platform channel (P-001).

## Positioning cautions

- Reminders are logistics (time/place/link), never clinical
  content — no prep questions that imply diagnosis.
- Sending messages on the physician's behalf needs explicit
  consent scope and an audit trail.

## Depends on

P-008 (agenda/appointments) and a notification channel
(WhatsApp Business API or email).
