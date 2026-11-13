+++
id = "P-066"
title = "Self-hosted LGTM observability backend behind Cloudflare"
status = "open"
created = "2026-11-13"
priority = "high"
labels = ["infra", "observability"]
links = ["T-099", "T-101", "T-103"]
+++

## Context

T-099 shipped the observability platform (OTLP push from the Worker). Backend decision: **self-hosted LGTM**. Constraint: must run on Cloudflare or integrate cleanly with the CF stack — the app deploys to Cloudflare Workers via Alchemy.

## Constraint analysis

Workers cannot host LGTM (Tempo/Loki/Mimir/Grafana are long-running stateful services). So "CF-compatible" means: **CF edge in front of self-hosted LGTM**, not LGTM on CF.

## Options

### A. LGTM docker-compose on a small VM + cloudflared tunnel (RECOMMENDED)

- Host: Hetzner CX22 (~€4/mo) or Fly.io machine (~$5-10/mo, Docker-native, anycast).
- Stack: `grafana/otel-lgtm` all-in-one image (single container: Grafana+Loki+Tempo+Prometheus+Alloy+otelcol) — simplest ops.
- Exposure: `cloudflared` tunnel → `otel.<ktbhealth-domain>` on the CF zone. TLS + Access (service token or IP rules) at the CF edge; origin stays private.
- Worker pushes OTLP/HTTP+JSON to `https://otel.<domain>/otlp` (TLS guard satisfied).
- CF-native baseline stays: Workers Logs+Traces enabled in wrangler for always-on signal even if LGTM is down.
- Pros: cheapest, full control, data residency, no vendor. Cons: we own uptime/patching/backups.

### B. LGTM on Fly.io with public TLS (no tunnel)

Same stack on Fly Machines + fly.toml; expose 443 directly with Fly TLS. Simpler (no tunnel daemon) but public origin needs app-level auth (OTLP headers — we support `OTEL_EXPORTER_OTLP_HEADERS`) or mTLS. CF in front optional via proxied DNS.

### C. CF-native only (no LGTM)

Workers Logs/Traces + CF Analytics/GraphQL + a CF Worker scraping `/api/metrics`. Zero new infra but: no long-term trace store, weak log querying, metrics retention limits, no Grafana. Doesn't meet "LGTM selfhost" decision — listed for completeness.

## Recommendation

**Option A** — `grafana/otel-lgtm` on a Hetzner/Fly VM behind a named cloudflared tunnel on the ktbhealth zone. Keeps data self-hosted, TLS+edge auth free via CF, zero ports open on the host, and the app's `OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.<domain>` just works (path `/v1/{logs,traces,metrics}` appended by config; otel-lgtm accepts OTLP on 4318 — may need a tiny path rewrite in the tunnel/ingress or per-signal endpoint config).

## Work breakdown (after approval)

1. `infra/lgtm/`: docker-compose (or fly.toml) + cloudflared config + otelcol tail_sampling processor (T-107) + Grafana provisioning (datasources, dashboards T-101, alert rules).
2. Provision host (needs user's Hetzner/Fly account — human step or credentials via cuora).
3. `cuora put cuora://ktb/otel-exporter-otlp-headers` if edge auth needs headers (CF Access service token) — uncomment cuora.toml line.
4. Set `OTEL_EXPORTER_OTLP_ENDPOINT` (+ optional per-signal) at deploy; uncomment nothing else — plain env via deploy shell or alchemy vars.
5. Deploy app (`infra:deploy`), verify `/api/ready` → all `otlp`, dashboards populate.

## Open questions

- Domain for otel endpoint (`otel.ktbhealth.dev`? check CF zone).
- Host preference: Hetzner (cheapest, manual) vs Fly.io (managed, docker-native).
- Grafana access: tunnel-only with CF Access (recommended) vs public.
