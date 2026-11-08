+++
id = "P-019"
title = "Patient register: full clinical-administrative record fields"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["directory"]
epic = "directory"
+++

## Idea

Today `patients` is a name-only table (`id`, `ownerId`, `name`,
`createdAt`). Grow it into a real register (cadastro): birth date,
phone/WhatsApp, email, insurance plan, allergies, chronic conditions,
emergency contact — the fields a Brazilian clinic actually collects at
intake.

## Likely scope

- Schema migration adding optional register fields (all nullable —
  existing rows stay valid)
- Patient create/edit form (extends the current name-only input in
  NewConsultationPage)
- Field-level LGPD care: mark which fields are health data vs.
  administrative; consent copy where required
- Validation: CPF optional, phone formatting, birth date sanity

## Why it matters

Name-only patients can't support reminders (P-009 needs a phone),
visit summaries (P-013 needs delivery channel), or any real clinic
workflow. The register is the foundation every other patient-facing
feature stands on.

## Positioning cautions

- Extra fields increase LGPD exposure — collect only what the consult
  flow needs; document purpose per field group.
- Allergies/chronic conditions shown to the physician are context,
  not advice — same documentation-only boundary as everything else.
- Keep fields optional; a consult must never be blocked by an
  incomplete register.

## Depends on

Nothing — schema + form work. Blocks P-009 (reminders need contact
data), P-020 (profile renders these fields), P-021.
