+++
id = "P-091"
title = "data.timeseries: append-only time-series pattern plugin (retention, downsampling, windowed aggregates)"
status = "open"
created = "2026-11-17"
owner = "devin"
priority = "low"
links = ["P-063", "P-089", "P-090"]
+++

## Problem

Append-only, high-volume, time-ordered data has no home in the persistence-pattern family: vitals streams, job/run metrics, transcription segment buffers, audit-volume events, eval result series. Drizzle tables work but every consumer hand-rolls retention, downsampling, and window queries; document-store (P-089) is wrong shape (per-doc JSON, JS-side filtering).

## Proposal

`plugins/data-timeseries` (`@ktb/data-timeseries`): `data.timeseries` pattern plugin — sibling to `data.documents` (P-089) and `data.kv` (P-090).

- Effect service `TimeSeries`: `append(series, point)`, `query(series, { from, to, agg? })`, `latest(series, n?)`.
- Point = `{ ts, value JSON, dims? }`; series declared as data (name, value schema via the same field DSL as documents, retention policy, downsample rules).
- Storage: one `ts_points` table (series_id, ts, value JSON, dims JSON) + `ts_series` registry — same migration-free philosophy as kv/documents.
- Retention: per-series TTL/downsample policy enforced by `sweep()` (scheduled via jobs infra if present); queries auto-select raw vs downsampled buckets.
- Aggregation: count/sum/avg/min/max + bucketed time buckets (`1m/5m/1h/1d`) computed in SQL over typed value columns where the value schema is scalar — JSON fallback for complex values.
- Tenant/owner scoping identical to kv/documents.

## Non-goals

- Not for relational or document data; not a metrics backend (no Prometheus export — that's telemetry's job).
- Build trigger: first consumer needing append-only retention — candidates P-063 (live transcription segments), job metrics, vitals/intake flows.

## Relations

- Family: P-089 (documents), P-090 (kv). Triggers off real consumers; do not build speculatively.
