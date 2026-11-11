+++
id = "P-022"
title = "Doctor registry: multi-practitioner accounts"
status = "open"
created = "2026-11-08"
priority = "med"
labels = ["directory"]
epic = "directory"
+++

## Idea

Today every account is a solo practitioner (`ownerId` = user). Add a
doctor registry: clinics/organizations with multiple practitioners,
each owning their patients and consults, plus org-level listing of
members.

## Likely scope

- `organizations` + `memberships` tables (or clinic_id on users);
  migration + drizzle schema
- Invite flow: doctor invites colleague by email → membership →
  scoped access (each doctor still owns their own patients/consults;
  org admin sees member list, not clinical data by default)
- Org settings page: members, roles (admin vs practitioner), remove
- AuthZ: extend permix rules (`@ktb/authz`) with org-scoped actions

## Why it matters

Real clinics have 2–50 doctors. Multi-user is the difference between
a pilot tool and a product a clinic can adopt — and it's prerequisite
for any shared scheduling (P-008) or history-sharing (P-006) across
practitioners.

## Positioning cautions

- Clinical data stays per-doctor by default — org membership must
  NOT silently grant access to consults/patients; P-006 consent
  scoping remains the only cross-doctor path.
- LGPD: clinic-as-controller vs doctor-as-controller needs a legal
  decision before building data-sharing features.

## Depends on

Auth model review; P-006 for any cross-practitioner data paths.
Independent of P-019–P-021/P-023 (those work per-doctor today).
