+++
id = "PL-010"
title = "Platform capabilities execution: apps/cli, jobs drivers, vault broker, machines"
status = "done"
created = "2026-11-18"
owner = "devin"
priority = "high"
links = ["T-120", "T-135", "P-097", "P-098", "PL-007", "PL-009"]
done = "2026-11-18"
+++

Execution plan for P-098 — platform capabilities added during P-097
review. Iteration counts per the approval: apps/cli 3, vault 5,
machines 3; jobs drivers ride WS-E in PL-009.

## Track A — apps/cli (3 iterations)

- [ ] A1: Composition root — extract shared bootstrap from
      `apps/web` into `@ktb/app-host` (registry load, port/adapter
      binding, plugin states); `apps/cli` skeleton on `effect/cli`
      (proven in `apps/eval`) with `--json` output and exit-code
      contract.
- [ ] A2: Verb discovery — map plugin contributions (commands/queries/
      fns) to CLI verbs; port eval verbs (generate/run/gate/report/
      calibrate/bench + suite discovery over `evals/`); delete
      `apps/eval`.
- [ ] A3: Broaden surface — jobs, vault/credentials, ai-provider/
      model/prompt, eval lab verbs; docs + `bun run cli` entry.

## Track B — jobs local queue (rides PL-009 WS-E)

- [ ] B1: `jobs-port` transport split — `JobsPort` contract vs drivers
      (`cf-queue` prod, `redis` dev); envelope/kind dispatch shared.
- [ ] B2: `compose.yaml` Redis (Valkey) service + local worker process
      (`bun run jobs:worker` or driver bin); dev docs; verify audio +
      eval jobs drain locally.

## Track C — vault secrets broker (5 iterations)

- [ ] C1: Schema + envelope encryption at rest (tenant DEK via
      EntityCipher), `secretRef` resolver port, constant-time compare,
      audit events on read/write.
- [ ] C2: CRUD + RBAC (`vault.secret`), masked reads, rotation API,
      no-raw-by-default fns.
- [ ] C3: Migrate consumers — ai-provider secretRef, email-smtp,
      captcha, machines authRef; merge `plugins/credentials` panel,
      redirect nav/exports compat.
- [ ] C4: Threat pass — access-log on reads, rate limits, no secret
      material in logs/telemetry/job payloads (payloadRef), ZK-tenant
      review.
- [ ] C5: Pen-test-lite + runbook + ADR (secrets-at-rest decision);
      docs.

## Track D — plugins/machines (3 iterations)

- [ ] D1: `machine` table + CRUD + RBAC + i18n; kinds
      local|ssh|vm; authRef → vault secretRef; capabilities[].
- [ ] D2: Health/last-seen probes; ai-provider endpoint wiring —
      child/ssh transports reference machines (inline fields kept as
      deprecated legacy for one release).
- [ ] D3: UI polish (platform panel, status badges), docs, seed data.

## Ordering

C1+C2 early (secretRef resolution unblocks ai-provider polish in
PL-009 WS-A); D1/D2 alongside WS-A; A1 can start any time (pure
refactor); B1/B2 with WS-E; C3 after consumers exist; A3 last.

Links: P-098 · P-097 · PL-009 · PL-007 · T-120 · T-135
