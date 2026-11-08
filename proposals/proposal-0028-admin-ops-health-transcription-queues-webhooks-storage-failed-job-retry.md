+++
id = "P-028"
title = "Admin ops health: transcription queues, webhooks, storage, failed-job retry"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["admin"]
epic = "admin"
+++

## Idea

Ops health views in `/admin`: a system-status page showing the moving
parts of the platform — transcription queue depth and job states
(P-015/P-016), webhook delivery success (Recall.ai, Stripe later),
Workers/Queue error signals, R2 storage usage, and recent failed jobs
with retry affordances.

## Likely scope

- `/admin/health` dashboard: queue depth, job status counts, last
  errors, storage usage
- Job detail: input metadata, status timeline, logs excerpt, manual
  retry
- Cheap data sources first (DB counts, Cloudflare analytics API);
  don't build a metrics pipeline for v1

## Why it matters

The async transcription pipeline (P-015/P-016) introduces the first
background-job failure modes the product has. Admin needs to see and
unstick them without `wrangler tail`.

## Positioning cautions

- Surface real signals, not vanity charts — queue depth and failure
  counts, not "uptime 99.9%".
- Manual retry must be idempotent; a stuck job shouldn't double-bill
  Whisper usage.

## Depends on

- P-015/P-016 audio + transcription pipeline, P-025 shell
