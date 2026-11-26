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

A peregrin-inspired, event-driven orchestration engine **inside product** —
simple model, emergent behavior — in TypeScript/Effect. Orchestrations become
stored, editable, serializable data; the consultation-transcript pipeline is
rewritten on it as proof.

## Model (peregrin core, ported)

Peregrin's insight: **edges are implicit**. A campaign is a set of nodes; each
node declares `on: [event types]` it reacts to and `emits: [event types]` it
produces. The graph emerges from event-type routing — no explicit links, no
DAG schema. Drivers are pluggable executors behind a small contract
(receive event + context → emit events).

- `OrchestrationDef` (JSON, versioned, DB-stored): `{ name, config, nodes[],
  event_schemas?, supersede?, annotations }`.
- `OrchestrationNode`: `{ id, kind, handler?, on[], emits[], mode: any|all,
  params, max_fires, until, retire_on, retry, circuit_breaker, poll?,
  fan_in?, state, input?, output? }` — direct port of peregrin's `Node`.
- `Event`: `{ type, payload, source, id, ts }` — typed, JSON-schema
  validated (`event.invalid` on mismatch), lifecycle events
  (`node.failed`, `node.exhausted`, `workflow.done/failed/cancel`).
- **Node kinds = peregrin's drivers**, adapted to product ports:
  - `stage` — existing `PipelineStage` code (the current pipeline impls)
  - `ai` — pure data: `{ template, model, input/output schema }` bound to
    `PlatformLlm` / `DecisionsPort` / `AsrPort`. No code → editable, reusable.
  - `gate` — condition on state → emits branch event types
  - `emit` — produce an event without work (routers, like peregrin's
    shell `emit:` maps)
  - `schedule`/`poll`, `subworkflow` (nested orchestration), `http` — later,
    same driver contract
- `OrchestratorRegistry` (Effect `Context.Service`, `PipelineRegistry` idiom):
  kind registry + def registry; plugins contribute defs via manifest, DB defs
  merge at boot.

## Engine

- In-process event bus per run (`Stream`/`Queue`), durable via two tables:
  `orchestration_runs` (state per node) + `orchestration_events` (log).
  Node fires scheduled through `jobs-port` — worker restart resumes from last
  completed node (peregrin's queue semantics, single-node scope — no
  multi-machine pairing, no hypha substrate).
- Runner port of `peregrin-engine/src/runner.rs`: match events → fire nodes,
  honor `retry`/`circuit_breaker`/`max_fires`/`until`/`retire_on`/`fan_in`/
  `supersede` (latest-wins dedupe of pending runs — directly useful for
  repeated consultation uploads).
- Effect mapping: `Schedule` → retry/circuit-breaker, `Stream` → event
  consumption, `Deferred` → fan-in joins, `Effect.timeout` → node timeouts.

## UI

Sidebar → Pipelines (data table) → create → editor. The stored def has **no
edges** — the editor *derives* the graph (edge exists where node A's `emits`
overlaps node B's `on`) and renders it in React Flow; editing `on`/`emits`
rewires implicitly. This is the peregrin model's real UI payoff: the canvas
shows emergent topology, not a hand-drawn DAG. Built as a `plugins/`
contribution; check `ui`/storybook first.

## Migration

Consultation-transcript ported last — must preserve PHI scrub gates, BYOK/ZK
guards, `RunRecord` audit, and eval/app parity (the P-118 invariant). The
def maps cleanly: stages → `stage`/`ai` nodes, `onError` → `retry`/`until`,
trace → per-node run records.

## Phases

1. Core: `OrchestrationDef`/`Node`/`Event` types + kind registry + runner over
   in-memory bus (peregrin-engine port, single process)
2. Durable runs: events/runs tables + `jobs-port` scheduling
3. DB def storage + registry merge + `OrchestratorRegistry` service
4. Pipelines editor (table + create + derived-graph canvas)
5. Consultation-transcript rewrite

## Non-goals

Multi-machine drivers, webhook ingress, agent campaigns, entity plane/pairing
(hypha's job) — product needs single-process, per-tenant orchestration, not
distributed campaigns.
