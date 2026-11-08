+++
id = "P-015"
title = "Consultation audio storage: R2 bucket + authenticated upload/fetch"
status = "open"
created = "2026-11-08"
priority = "high"
labels = ["audio-pipeline"]
epic = "audio-pipeline"
+++

## Idea

Persist the session audio the browser already records (WhisperTranscriber MediaRecorder sidecar) to Cloudflare R2, linked to the consultation. Today the blob is only kept in memory for the low-confidence server fallback and then discarded — the raw evidence is lost.

## Likely scope

- R2 bucket provisioned via Alchemy (infra/cloudflare + apps/web/alchemy.run.ts binding)
- Authenticated upload route (POST audio → R2 key per consultation/user) and fetch route for playback/download
- D1 linkage: consultations.audio_key (or side table) + upload status
- Retention/deletion policy aligned with LGPD — audio is biometric-adjacent voice data; define TTL or delete-on-consult-deletion

## Why it matters

Raw audio is the audit trail behind the transcript: disputes, poorRecognition re-runs, and the bigger-model pass (P-016) all need the source. Also enables future adapters (P-007 upload/desktop) to reuse the same storage path.

## Positioning cautions

- Voice recordings are sensitive personal data — storage must be opt-in per consult flow or clearly disclosed; never uploaded silently.
- Encryption at rest via R2 defaults; no public bucket URLs — fetch only through authenticated Worker routes.
- Retention must be documented; consider auto-expiry (e.g. 90 days) unless physician marks keep.

## Depends on

Nothing — foundational. Blocks P-016 (queue needs stored audio) and P-018 (playback UI).
