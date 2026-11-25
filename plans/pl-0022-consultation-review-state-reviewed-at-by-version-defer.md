+++
id = "PL-022"
title = "Consultation review state: reviewed_at/by/version + defer (P-014)"
status = "open"
created = "2026-11-24"
owner = "devin"
priority = "high"
labels = ["review-queue", "product"]
links = ["P-014"]
+++

## Goal

Give the physician an explicit, persisted "reviewed" state per
consultation — the missing half of P-014's review queue. The list
already shows the machine's guess (low-confidence badge); this adds
the human act: who approved the anamnesis, when, and which version.

## Gaps found vs. first draft (code review)

- `consultations.review` exists but is a **dead legacy column**
  ("second-opinion review removed") — do NOT reuse; new columns.
- The anamnesis is **versioned and hash-chained** (`anamnesis_version`,
  record_versions). Approval must bind to a version —
  `reviewed_version` — or an edited draft keeps the "reviewed" badge
  dishonestly.
- `TablePage` selection already exists — but P-014's own caution bans
  blind bulk-approve, so "batch" = **approve-and-next-pending**
  navigation, not a bulk mutation fn.
- Audit is a direct `audit()` call inside the service (ids only — PHI
  never in detail), not a CQRS event.
- ZK rows: review columns are metadata, not PHI — approval works on
  ciphered rows unchanged.

## Phases

- [ ] **F1 Schema** — **two separate approvals** (intent check):
  `transcript_reviewed_at` + `transcript_reviewed_by`, and
  `anamnesis_reviewed_at` + `anamnesis_reviewed_by` +
  `anamnesis_reviewed_version` (links `anamnesis_version`), plus
  `deferred_until` (ts, NULL=not deferred, record-level). Transcript
  approval binds no version (the column is not versioned — it is raw
  evidence); anamnesis approval binds the exact version reviewed.
  `bun run db:generate` + `db:migrate:local` + ERD regen (docs repo
  needs same-name branch for the erd CI job).
- [ ] **F2 Service + fns** — `Consultations.reviewTranscript(id)` /
  `approveAnamnesis(id)` / `defer(id, until)` / `clearDefer(id)`;
  authz via `requirePrincipalUser` + `consultationScope` (same as
  `save`); audit actions `consultations.reviewTranscript` +
  `consultations.approveAnamnesis` (ids + version only — PHI never in
  detail). Fns `reviewTranscriptFn`, `approveAnamnesisFn`,
  `deferConsultationFn` with `effect/Schema` wire validation.
- [ ] **F3 List** — `ConsultationSummary` + query carry the new
  fields; facet "revisão" (pendente / transcrição conferida /
  revisada / adiada); badges: "transcrição conferida" (info) when
  only the transcript is confirmed, "revisada v⟨n⟩" (green) when the
  anamnesis is approved, "adiada até <date>"; stats "pendentes" =
  `anamnesis_reviewed_at IS NULL AND (deferred_until IS NULL OR
  deferred_until <= now)`. Keep low-confidence badge — orthogonal
  signal.
- [ ] **F4 Detail** — two acts (intent check): "Conferir
  transcrição" on the transcript card + "Aprovar anamnese" on the
  anamnesis card/header. Approved shows "por <user> em <date> · v<n>"
  per artifact; stale-version state "nova versão v<m> não revisada"
  when `anamnesis_version > anamnesis_reviewed_version` (edit keeps
  the flag + warns — never loses who approved); defer via `calendar`
  date picker; "Aprovar e próxima pendente" navigates to the next
  consult with unapproved anamnesis — in this delivery.
- [ ] **F5 i18n + gate** — ~12 keys ×3 locales + gen manifest/types;
  100% coverage on touched files; `bun run check`; PR.

## Out of scope

Dedicated queue page (facet preset covers it), bulk blind-approve fn
(P-014 caution), un-approve action (stale-version badge suffices — a
new approve re-stamps), dropping the legacy `review` column (separate
cleanup), admin/portal surfaces.

## Defer semantics

Derived, no state machine: `reviewed_at IS NULL AND deferred_until >
now` → "adiada"; when the date passes the row reappears as pending.
Defer is an **arbitrary date** (intent check) via the ui `calendar`
component (convention: agenda/patients date inputs). Deferred applies
to the whole record — `anamnesis_reviewed_at IS NULL AND
deferred_until > now` → "adiada"; when the date passes the row
reappears as pending. No cron, no scheduler.
