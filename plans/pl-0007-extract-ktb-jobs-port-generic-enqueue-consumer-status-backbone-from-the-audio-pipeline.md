+++
id = "PL-007"
title = "Extract @ktb/jobs port: generic enqueue/consumer/status backbone from the audio pipeline"
status = "done"
created = "2026-11-15"
priority = "med"
links = ["P-082", "PL-006"]
done = "2026-11-15"
+++

Prerequisite hardening for PL-006 Phase 4 (theme asset jobs) — and every future async workload. Today the only job pipeline is hand-rolled in `apps/web/src/lib/audio-jobs.ts` (queue consumer hardcoded to audio), `AudioJobMessage` in `@ktb/server-fns`, `AUDIO_QUEUE`/`AUDIO_BUCKET` bindings, state on the consultations row, manual admin retry. `plugins/jobs` is console UI only — it renders projections, runs nothing.

## Steps

1. **Port** (`packages/jobs`, `@ktb/jobs`): `Jobs` Effect service — `enqueue(kind, payload)` port + `JobStatus`/`JobRecord` schema. Message envelope `{ kind, payload, id, tenantId? }` validated by effect/Schema at the consumer edge (same trust boundary as `AudioJobMessageSchema`).
2. **Status projection**: `job_runs` table (packages/db schema) — id, kind, status queued|processing|done|failed, payload ref (no PHI/blob inline), error, timestamps, tenant scope. This is what `plugins/jobs` console feed reads — decouples job state from domain rows (`audioStatus` becomes a projection/cache, not the source of truth; migration path noted, not required v1).
3. **Consumer registry**: kind → handler map; the queue consumer dispatches by `kind`. Handlers are Effect services registered per plugin (theme assets, future exporters).
4. **Migrate audio**: `audio-jobs.ts` becomes a `transcription.audio` kind handler behind the port; `consultations.$id.audio.ts` upload route enqueues via the port instead of the raw queue. Proves the abstraction with the existing workload — no theme code yet.
5. **Bindings**: queue/bucket stay env-composed at the edge (wrangler producer/consumer bindings); the port hides provider specifics per ADR-0001. `ktb-theme-assets` queue (PL-006 decision) registers as another producer/consumer pair.
6. **Tests**: envelope schema, dispatch, retry/failure marking, tenant scoping; 100% coverage gate per package.

## Non-goals

- No workflow DAG / scheduling / cron — that's eval-platform territory (P-007x series).
- No cross-repo job broker; Cloudflare Queues stays the transport.
- Audio row-state migration to `job_runs` is optional follow-up, not blocking.

## Relationship to PL-006

PL-006 Phase 4 (theme asset pipeline) consumes this port. Phases 1–3, 5–7 of PL-006 do not depend on it — only the upload/job mechanics do.
