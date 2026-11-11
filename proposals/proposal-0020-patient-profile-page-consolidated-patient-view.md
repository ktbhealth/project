+++
id = "P-020"
title = "Patient profile page: consolidated patient view"
status = "open"
created = "2026-11-08"
priority = "med"
labels = ["directory"]
epic = "directory"
+++

## Idea

A dedicated `/patients/$id` page: register fields (P-019), upcoming
appointments (P-008), recent consults, quick actions (new consult,
message, export). The patient becomes a first-class entity instead of
a name inside a dropdown.

## Likely scope

- Route + page component in plugins/patients (or consultations)
- Sections: register data (editable), consult history list, agenda
  slice, notes (P-011 feeds in later)
- Entry points: patient select on consult pages links to profile;
  patients list rows link through
- Owner-scoped authz identical to existing patient endpoints

## Why it matters

Today a patient is just a string on a consultation. A profile makes
the longitudinal relationship visible — the retention surface that
turns a scribe into the doctor's patient memory.

## Positioning cautions

- Profile shows recorded facts only — no inferred risk scores or
  care suggestions.
- Editing register fields is physician/admin action; audit-log
  changes to health-relevant fields later if required.

## Depends on

P-019 (fields to display/edit), P-004 (timeline UI can embed here),
P-008 optional (appointments section degrades gracefully).
