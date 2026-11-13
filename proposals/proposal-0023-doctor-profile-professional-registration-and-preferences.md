+++
id = "P-023"
title = "Doctor profile: professional registration and preferences"
status = "open"
created = "2026-11-08"
priority = "med"
labels = ["directory", "product"]
epic = "directory"
+++

## Idea

A doctor profile/settings area: professional data (name, CRM, specialty,
clinic), notification preferences, consultation defaults (language,
output template), and account/security settings. Works for the solo
doctor today; becomes the member profile under P-022 later.

## Likely scope

- `/settings` or `/profile` route: profile form + preferences
- Users table extension (display name, CRM, specialty, clinic —
  professional data, not clinical)
- Preferences consumed by existing flows: default transcription
  language, anamnesis template, reminder timing (P-009)
- Security section: sessions, password change — whatever better-auth
  already exposes

## Why it matters

The doctor is the customer; a profile makes the product feel like
*their* workspace and is where per-doctor defaults live. Cheap to
build, high perceived polish.

## Positioning cautions

- Professional data only — no patient data on this page.
- Specialty/CRM fields are descriptive, not verified credentials —
  don't imply platform certification.

## Depends on

Nothing structural. P-022 later adds org context to the same page.
