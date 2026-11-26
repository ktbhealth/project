+++
id = "P-120"
title = "Event-driven orchestrator: serializable pipelines with AI nodes"
status = "open"
created = "2026-11-26"
owner = "devin"
priority = "med"
labels = ["orchestration", "pipelines"]
+++

## Goal

General event-driven orchestration layer in product: pipelines become stored, editable, serializable data; the consultation-transcript pipeline is rewritten on it as proof.

## Model

- `OrchestrationDef` (JSON, versioned): triggers/events → **general nodes** → routing. Nodes are a typed registry — `stage` (existing `PipelineStage` code), `ai` (prompt/model/IO-schema as data, bound to `PlatformLlm`/`DecisionsPort`/`AsrPort` ports), `gate` (condition on state), `emit` (event out), extensible by plugins.
- `OrchestrationRun`: durable record — per-node status/trace persisted after each step; resume via `jobs-port` consumer; `onError` policies carry over from `@ktb/pipeline`.
- `OrchestratorRegistry` service (Effect `Context.Service`, same idiom as `PipelineRegistry`); plugins contribute defs via manifest, DB-stored defs merge at boot.

AI nodes are one node kind, not the model — they exist because they make nodes data-driven (no code) and reusable across orchestrations. Deterministic work stays code-backed `stage` nodes.

## UI

Sidebar → Pipelines (data table) → create → editor. Linear chain first; DAG only if the model goes event-routed. Built as a `plugins/` contribution per repo conventions; check `ui`/storybook before building components.

## Migration

Consultation-transcript ported last — must preserve PHI scrub gates, BYOK/ZK guards, `RunRecord` audit, and eval/app parity (the P-118 invariant).

## Phases

1. Durable runs on `jobs-port` (resumability)
2. DB def storage + registry merge
3. Pipelines editor (table + create + linear editor)
4. Model generalization (events/routing) only if a second domain demands it
5. Consultation-transcript rewrite

## Non-goals

Multi-machine drivers, webhook ingress, agent campaigns — that is peregrin's job, not product infra.
