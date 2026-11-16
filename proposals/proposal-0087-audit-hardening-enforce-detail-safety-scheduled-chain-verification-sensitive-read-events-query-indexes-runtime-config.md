+++
id = "P-087"
title = "Audit hardening: enforce detail safety, scheduled chain verification, sensitive-read events, query indexes, runtime config"
status = "in-progress"
created = "2026-11-16"
priority = "med"
+++

# Audit hardening: enforce detail safety, scheduled chain verification, sensitive-read events, query indexes, runtime config

Status: **accepted** · Created: 2026-11-16

## Problem

Post-P-086 review of `plugins/audit` surfaced five concrete weaknesses:

1. **`detail` safety is convention, not enforcement.** `serializeDetail` only caps at 4000 chars — a caller can stuff PHI/secrets into the JSON and nothing stops it.
2. **`verifyAuditChain` is unwired.** It exists (repo fn + `Audit.adminVerify`) but nothing calls it outside tests — tampering is never actually detected in production.
3. **No sensitive-read events.** Every `audit()` call is on a mutation path; who *viewed* a patient chart or consultation (the LGPD access-log case) is invisible. The patient feed (P-086) was designed to show these once emitted.
4. **Read-event volume unaddressed.** Hash-chained inserts are write-amplified; naive read auditing would flood the log (e.g. `getLiveConsultationFn` polls every 1.5s).
5. **No runtime configuration.** Coalescing windows / kill-switches would be hardcoded.

## Proposal

### D1 — Enforce detail safety in `serializeDetail`

Recursive sanitizer applied before the 4000-char cap:

- Drop any object key matching a case-insensitive substring blocklist: `name, email, phone, cpf, cnpj, birth, dob, address, transcript, anamnesis, diagnos, prescri, exam, note, content, text, body, message, token, secret, password, key, dek, payload, filename`. Substring (not `\b`) matching so camelCase like `patientName` is caught. Over-redaction is acceptable; under-redaction is not.
- Recurse into nested objects/arrays; cap depth (4), array length (20), string length (200).
- Non-JSON-safe values coerced/dropped; empty result → `null` detail.
- **Call-site audit**: grep every `audit({ detail: … })` and confirm no legitimate key is dropped (`patientId`, `ownerId`, `hadAudio`, `engine`, `via`, counts must survive — none match the blocklist).

### D2 — Sensitive-read events via action-suffix convention

New actions follow `<domain>.<noun>.read` (and `.export` for bulk downloads). Emission at **service-method level** so fns, loaders, API routes, and portal paths are all covered by one call site:

| Site | Action | Notes |
|---|---|---|
| `Patients.get` | `patients.read` | chart, edit, pdf, export, deletion-impact, extractAnamnesis, clinician loader |
| `Consultations.get` | `consultations.read` | detail + audio/pdf/export authz paths; poll flood handled by D3 |
| `Consultations.history` / `historyRows` | `consultations.history.read` | prior-visit PHI summaries/raw envelopes |
| `PatientPortal.consultation` | `consultations.read` + `detail.via:"portal"` | patient reading own consult — feeds P-086 patient log |
| `GET /api/consultations/:id/audio` | `consultations.audio.read` | recording download |
| `GET /api/consultations/:id/pdf` | `consultations.export` | rendered PHI PDF |
| `GET /api/patients/:id/export` | `patients.export` | FHIR/openEHR bulk export — highest sensitivity |

Explicitly skipped (documented in code): `getLive` (1.5s mechanical poll), all list endpoints, `portalFn` home, `dekEnvelopes`, `patientLinksFn`, `portalReadersFn` — high volume, low accountability value.

- `tenantId` from the **row** (`row.tenantId`), matching the `patients.update` precedent — cross-owner admin/tenant reads scope correctly.
- `detail.patientId` included on consultation reads so `listPatientAuditEntries` joins them into the patient feed (live consult join + deleted-consult fallback already handle it).
- `patient-portal` gains `optionalDeps: [auditServicesPlugin.id]` (currently absent).
- `audit()` is already a no-op when audit is absent and `record` never fails — safe unconditionally.

### D3 — Read coalescing inside `Audit.record`

Convention: any action ending in `.read` is coalesced — before insert, skip when an identical `(actorUserId, action, targetType, targetId)` row exists within the window. Default **60s**, env-configurable, `0` disables.

- Gives "at most one read event per actor/target/minute" — sufficient for accountability, kills poll/list floods.
- New composite index `admin_audit_dedupe_idx (actor_user_id, action, target_type, target_id, created_at)` serves the lookup. Migration via `db:generate`.
- Applies uniformly to all `.read` actions — no new API surface; emission sites just call `audit()`.

### D4 — Scheduled chain verification (first cron)

- `apps/web/src/server.ts` gains a `scheduled()` export; cron trigger added to the deploy config (Alchemy/wrangler — agent verifies which). Daily off-peak schedule.
- Handler runs `verifyAuditChain(db)` under `runAsTenantId(null, …)`:
  - `verified`/`empty` → structured log only (no row spam).
  - `tampered` → error log + append a system row `audit.chain.tampered` (`actorUserId` null) so the tamper event itself is chained into the log.
- On-demand: new `auditVerifyFn` server fn (admin-gated via `Audit.adminVerify`) + "Verify chain" button on the platform `/platform/audit` page showing the result — covers dev (no cron) and ad-hoc checks.
- Known limitation documented: full-table SHA-256 walk is O(n); fine at current volume, chunking is future work.

### D5 — Runtime configuration

Via the repo's env/config mechanism (NOT `process.env` — arch lint bans it in plugins; agent finds the sanctioned path, likely `@ktb/env` or platform config):

- `AUDIT_READ_COALESCE_MS` — D3 window (default 60000).
- `AUDIT_READS` — `"0"` disables read-event recording entirely (kill switch for volume incidents); mutation events unaffected.
- `AUDIT_ENABLED` — `"0"` disables all recording (global kill switch).

## Boundaries

- Same `admin_audit` table + hash chain; no parallel store, no schema columns beyond the dedupe index.
- Read events carry the same detail rules (ids/flags/`via` only) — D1 enforcement applies to them too.
- No PHI in any new event; patient feed gains read rows automatically through the existing join/fallback.
- Cron is additive infra — `fetch`/`queue` handlers untouched.

## Open questions

- Exact env-access mechanism inside `AuditLive` (resolved during implementation).
- Whether `consultations.list`/`adminList` (which ship full PHI columns) deserve a coarse `*.list.read` event later — deferred; lists are high-volume and the detail reads already capture deliberate access.
