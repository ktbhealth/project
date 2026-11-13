+++
id = "P-067"
title = "LGTM edge + ops decisions: tunnel mode, alert receiver, endpoint env"
status = "done"
created = "2026-11-13"
priority = "med"
labels = ["infra", "observability"]
links = ["T-101", "T-109", "P-066"]
+++

## Context

P-066 chose self-hosted LGTM behind Cloudflare; infra/lgtm/ is coded (compose, otelcol tail_sampling, provisioning, dashboards, alerts, runbook). T-109 tracks provisioning. Three decisions remain before the stack goes live.

## 1. Tunnel mode + hostname (DECIDE)

Two supported modes (both documented in infra/lgtm/README.md step 5):

- **Locally-managed tunnel** — `cloudflared tunnel create` on the host, ingress rules in `infra/lgtm/cloudflared/config.yml` (in git). Pros: config-as-code, reviewable, reproducible. Cons: cert.pem + credentials JSON live on host.
- **Remotely-managed tunnel (token)** — tunnel created in CF dashboard, `CLOUDFLARED_TUNNEL_TOKEN` in `.env`, optional compose service. Pros: zero host setup beyond token, ingress editable in dashboard. Cons: config not in git.

**Recommendation: locally-managed** — ingress is infra code, consistent with the repo's everything-as-code approach; token mode remains documented fallback.

Hostname proposal: `otel.ktbhealth.workers.dev` is taken by the worker pattern — use the real zone: `otel.<prod-domain>` and `grafana.<prod-domain>` (confirm zone in CF dashboard; likely `ktbhealth.com`/`ktbhealth.dev`). OTLP endpoint = `https://otel.<domain>` (no path — config appends `/v1/<signal>`).

## 2. Alert receiver (DECIDE)

`provisioning/alerting/alerting.yaml` has a placeholder email contact point. Options:

- **ntfy** — org already self-hosts ntfy (container `ntfy` running locally). Publish alerts to an ntfy topic → phone push. Cheapest, zero new infra, but ntfy is currently local-only — would need exposing via tunnel too (same pattern as LGTM).
- **SMTP email** — `GF_SMTP_*` env + real address; simplest, no new infra, but email alerting is slow/noisy.
- **Webhook (Slack/Discord/Gotify)** — swap contact point `receivers` to a webhook URL; one-line change, needs a workspace.

**Recommendation: ntfy** — already operated by the org, mobile push is right for a 5-rule alert set; expose `ntfy.<domain>` through the same cloudflared tunnel (one extra ingress line). Fallback: email until ntfy is public.

## 3. Endpoint env strategy (DECIDE)

`OTEL_EXPORTER_OTLP_ENDPOINT` is plain (non-secret) config but must differ per environment eventually (dev could point at local otelcol, prod at LGTM). Options:

- **Deploy-time shell env** — export before `infra:deploy`; alchemy.run.ts already forwards it when present. Zero code, but invisible in repo.
- **Committed per-stage value in alchemy.run.ts** — e.g. stage-conditional `OTEL_EXPORTER_OTLP_ENDPOINT: "https://otel.<domain>"` for prod stage only. Self-documenting, survives shell-env forgetfulness; endpoint isn't secret.
- **Cuora secret** — unnecessary; endpoint is not sensitive (headers carry auth if needed).

**Recommendation: committed stage-conditional in alchemy.run.ts** — endpoint is config, not secret; making it explicit per stage prevents "deployed without endpoint" drift. `OTEL_EXPORTER_OTLP_HEADERS` stays cuora-gated if edge auth is enabled.

## Work after decisions

1. T-109 provisioning (host, compose, tunnel, DNS).
2. Apply decisions: ingress hostname(s), alert contact point (+ntfy tunnel ingress if chosen), alchemy.run.ts endpoint line.
3. `cuora put` any new secrets (none expected — endpoint is plain env).
4. `infra:deploy`, verify `/api/ready` → all `otlp`, dashboards populate, test one alert.
