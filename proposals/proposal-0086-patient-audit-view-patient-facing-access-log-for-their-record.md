+++
id = "P-086"
title = "Patient audit view: patient-facing access log for their record"
status = "done"
created = "2026-11-16"
priority = "med"
done = "2026-11-16"
+++

## Problem

P-080 gave staff a tenant activity feed, but patients have no visibility into what happened to their record — the LGPD accountability gap P-030 originally named ("access log for sensitive reads"). Today `admin_audit` records mutations only; patients can't see who created/updated/deleted their record or when.

## Proposal (v1)

A patient-facing "access log" inside the patient portal: rows from `admin_audit` scoped to the patient's own record.

- **Scope**: v1 shows mutation events that name the patient record — `targetType="patient" AND targetId=<patients.id>` (patients.create/update/delete) plus consultation events whose `detail.patientId` matches (anamnesis.attach, save — audit detail already carries `patientId` for those). Read/access events are explicitly out of scope for v1 (volume + policy decision, separate proposal if wanted).
- **Read path**: new `Audit.patientList({ patientId, limit })` — resolves the bound patient identity via the patient-portal auth port (exact mechanism TBD by exploration: patient_access grant / portal session), fails `Forbidden` when the principal can't see that patient. Platform-global and other patients' rows never leak.
- **UI**: page under the patient portal instance (`<basePath>/access-log` or `/activity`), reusing `AuditPage` with a patient-safe title/empty copy; actor shown as staff display name/email already stored on the row.
- **Emission gaps**: audit `detail` on consultation events is inconsistent about `patientId` — v1 adds `patientId` to the detail of consultation events that lack it (audio.attach, audio.status, consent.record, create/save where missing) so the patient feed can join on it. Still ids only — no PHI.

## Boundaries

- Same `admin_audit` table + hash chain; no parallel patient-event store.
- Patient-facing copy is factual ("record updated"), never interpretive — positioning rule holds.
- Patient auth must not weaken: the read path reuses whatever gate patient-portal pages already use; no new trust boundary.

## Open questions

- Exact patient-identity mechanism in effect context (portal_keys session vs patient_access grant vs patients.ownerId) — needs code exploration before design locks.
- Whether consultation events should get a first-class `patientId` column instead of JSON detail (schema change vs JSON query — v1 assumes JSON detail + app-level filter).
- Secretaries/guardians with patient_access grants: do they see the same log as the patient? (Probably yes — they act for the patient.)
