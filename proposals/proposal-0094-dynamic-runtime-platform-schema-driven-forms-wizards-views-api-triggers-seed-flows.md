+++
id = "P-094"
title = "Dynamic runtime platform: schema-driven forms/wizards/views/api/triggers/seed/flows"
status = "open"
created = "2026-11-18"
priority = "high"
labels = ["runtime"]
links = ["P-065"]
+++

# Dynamic runtime platform

Stack of 8 PRs (#127-#137) implementing the "definitions are data, plugins plug behavior" layer:

- `@ktb/json-schema` — draft-2020-12 subset contract (packages/json-schema)
- `@ktb/dsl` — shared condition/template/path/grammar vocabulary (packages/dsl)
- `@ktb/seed` — deterministic schema-driven fixtures (packages/seed)
- `@ktb/forms` — schema→input renderer + `form.def` (plugins/forms)
- `@ktb/wizards` — guided flows, runs, `/wizards/*` + `/platform/wizards` (plugins/wizards)
- `@ktb/views` — `view.def` → table/cards/detail at `/v/*` (plugins/views)
- `@ktb/api` — `api.expose` default-deny → `/api/documents/*` CRUD (plugins/api)
- `@ktb/triggers` — `trigger.def` write-reactions via EventBus (plugins/triggers)
- `@ktb/flows` — `flow.def` document state machines (plugins/flows)

Audit: 3 rounds, ~25 findings fixed (forged-def authz bypass, completion races, ReDoS, prototype-chain, enablement gaps). Residuals → T-150.

## Next
- Dogfood: first real emitters (patient-intake wizard + form/view/api.expose in an existing plugin)
- Deferred: visual wizard editor, DocumentStore.find SQL pushdown (T-129), seed consumer (T-042/T-092), hook-registry factory
