+++
id = "P-051"
title = "Restart-persistent local queue for dev:node (SQLite-backed)"
status = "done"
created = "2026-11-10"
priority = "med"
labels = ["devx"]
+++

## Problem

`dev:node` local infra persists everywhere except the queue: `makeLocalAudioQueue` (apps/web/src/lib/node-audio.ts) delivers in-process, so messages in flight are lost on restart; only exhausted retries reach the JSON dead-letter dir. Every other piece (SQLite app DB, FS audio bucket, .dev.vars) already survives restarts.

## Proposal

SQLite-backed queue in `apps/web/data/queue.db` (isolated file — no drizzle migration conflicts, gitignored via `data/`):

```sql
CREATE TABLE IF NOT EXISTS jobs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  queue TEXT NOT NULL,
  body TEXT NOT NULL,                    -- JSON message
  status TEXT NOT NULL DEFAULT 'pending',-- pending|processing|done|failed
  attempts INTEGER NOT NULL DEFAULT 0,
  max_attempts INTEGER NOT NULL DEFAULT 3,
  run_at INTEGER NOT NULL,               -- epoch ms; backoff scheduling
  last_error TEXT,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
CREATE INDEX idx_jobs_poll ON jobs(status, run_at);
```

- `send()` → INSERT pending row, return immediately (same contract as CF Queue binding).
- Poller: in-process interval (~250ms) claims due rows atomically (`UPDATE … WHERE id = (SELECT … LIMIT 1) RETURNING *`), delivers batch to the existing `handleAudioQueueBatch` consumer — real batching like CF Queues.
- Retry: failure → attempts++, run_at = now + backoff, back to pending; attempts >= max → `failed`. DLQ = `SELECT * FROM jobs WHERE status='failed'` — inspectable + replayable, replaces the JSON dir.
- Crash recovery: on boot, `processing` rows reset to `pending`.
- Wiring: replaces `makeLocalAudioQueue` in `node-cloudflare-workers.ts`; `node-audio.ts` keeps only the bucket; in-process impl deleted (knip).

## Fit

Pure deploy-edge glue in apps/web (exempt zone), zero new deps (reuse `makeSqliteDatabase`/`better-sqlite3`), no ports/packages touched, workerd dev path unchanged.

## Cost / gain

~150 lines + tests (claim atomicity, backoff, crash recovery, DLQ). Gain: dev:node survives restarts mid-transcription — the last volatile piece of the local stack.
