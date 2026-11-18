+++
id = "P-098"
title = "Platform capabilities: apps/cli composition root, local job queue, vault secrets broker, machines registry"
status = "open"
created = "2026-11-18"
owner = "devin"
priority = "high"
labels = ["architecture-platform", "eval"]
links = ["T-120", "T-135", "P-097", "PL-007", "PL-009"]
+++

## Context

P-097 (eval platform unification + AI capability plugins) was approved.
Review surfaced four additional platform capabilities that the eval
work depends on or generalizes. This proposal captures them; PL-009
remains the eval execution plan, PL-010 (this proposal's plan) tracks
these workstreams.

## 1. `apps/cli` — general plugin CLI composition root

`apps/web` is the HTTP composition root: it loads the plugin registry,
binds ports/adapters, and serves routes/fns/panels. `apps/cli` is the
same composition root **headless** — same registry, same port/adapter
selection, no HTTP server — exposing a uniform CLI surface over plugin
contributions (commands, queries, fns). `apps/eval` is deleted: eval's
`generate|run|gate|report|calibrate|bench` become CLI verbs available
through the general surface rather than a bespoke app.

- Iterations: 3.
  1. Composition root: registry load + port binding parity with
     `apps/web` (shared bootstrap extracted to `@ktb/app-host`), CLI
     framework (`effect/cli` already proven in `apps/eval`), verb
     discovery from plugin contributions.
  2. Eval verbs: port `apps/eval` commands onto the general surface;
     delete `apps/eval`; suite discovery keeps working from `evals/`.
  3. Broader plugin verbs: expose high-value plugin fns (jobs,
     credentials/vault, ai-provider/model/prompt, eval lab) as CLI
     commands; docs.

## 2. Jobs: local + worker queue drivers

`@ktb/jobs-port` (PL-007) owns the contract; transport is Cloudflare
Queue at the edge today. Add the missing local leg so the same job
kinds run in dev:

- `jobs` driver for **local dev**: Redis (Valkey) in `compose.yaml`
  + a local worker process draining it — same envelope/kind dispatch
  as the CF consumer. `docker compose up` gives a working queue with
  no cloud account.
- Iterations folded into WS-E (runner resilience): port-adapter split
  (`JobsPort` enqueue/consume vs transport drivers `cf-queue` /
  `redis`), compose service, dev docs.

## 3. `plugins/vault` absorbs credentials — the secrets broker

Decision: **vault becomes THE secrets/credentials broker** —
provider API keys, SMTP creds, `secretRef` material, platform API
keys. `plugins/credentials`' panel/functionality merges into vault;
`secretRef: "credential:<id>"` refs (ai_endpoint and friends) resolve
through vault. Vault keeps its existing key-lifecycle role
(clinicKeys/entityDeks/portalKeys) — secrets storage is a new domain
inside the same plugin, sharing the crypto/audit machinery.

- Security posture (5 iterations):
  1. Schema + envelope encryption at rest (per-tenant DEK via existing
     EntityCipher port), `secretRef` resolver port with constant-time
     compare + audit events.
  2. CRUD + RBAC (`vault.secret` resource), masked reads, rotation
     API, never-return-raw-by-default fns.
  3. Consumers migrated: ai-provider secretRef, email-smtp, captcha,
     ai-endpoints→machines; delete `plugins/credentials` panel,
     redirect nav.
  4. Threat pass: access-log on reads, rate limits, no secret material
     in logs/telemetry/jobs payloads (payloadRef indirection), ZK-tenant
     interaction review.
  5. Pen-test-lite + docs: red-team checklist, runbook, ADR for
     secrets-at-rest decision.

## 4. `plugins/machines` — machine registry for ACP

ACP agents run on machines/shells/VMs; `ai_endpoint` already carries
`transport`/`agentCmd`/`sshHost` ad-hoc. `plugins/machines` owns the
**inventory + health**, nothing more (decided scope):

- `machine` table: `{kind: local|ssh|vm, endpoint/host, authRef
  (vault secretRef), capabilities[], status, lastSeenAt}` + CRUD UI +
  RBAC + health checks.
- Consumers: ai-provider endpoints reference a machine for
  child/ssh transports; the ACP bridge/agent-link flow resolves
  through it. Agent sessions/execution stay in ai-adapter-node —
  machines plugin does not spawn or monitor agents.
- Iterations: 3 — (1) schema+CRUD+RBAC, (2) health/last-seen +
  ai-provider wiring, (3) UI polish + docs.

## Relationship to P-097/PL-009

- WS-E (runner resilience) now includes the Redis local queue driver.
- WS-A (`ai-provider`) consumes vault secretRefs and machines registry
  — order: vault secrets domain and machines registry land before or
  alongside WS-A polish.
- WS-C (`ai-prompt`) unaffected.
- `apps/cli` becomes the home for eval verbs; `apps/eval` deletion is
  the last iteration of the apps/cli track.

## Non-goals

- No agent execution/session management in machines (registry+health
  only).
- No secrets in env vars going forward — vault is the store; env stays
  bootstrap-only.
- No change to CF Queue prod transport — local driver parity only.

## Risks

- **apps/cli scope creep** — it must reuse `@ktb/app-host` bootstrap,
  not fork it; verb surface stays contribution-driven.
- **Vault merge churn** — credentials panel consumers (eval lab
  pickers) need redirect-compatible fns during transition.
- **Redis as new infra dep** — compose-only, never prod; prod stays
  CF Queues.
- **Secret material in job payloads** — enforce payloadRef pattern;
  audit in iteration 4.

## Acceptance criteria

- `bun run cli <verb>` (name TBD) runs eval generate/run/gate/report
  against the same registry as the web app; `apps/eval` deleted.
- `docker compose up` provides a working local job queue; eval runs
  and audio jobs drain locally without cloud creds.
- All provider/API-key secrets resolve through vault; `plugins/
  credentials` merged; reads audited; raw secrets never returned by
  default.
- Machines CRUD + health live; ai-provider ssh/child endpoints
  reference machines instead of inline host fields (fields kept as
  legacy/deprecated one release).

## Links

P-097 · PL-009 · PL-007 (jobs-port) · ADR-0001 · ADR-0004 · T-120 ·
T-135
