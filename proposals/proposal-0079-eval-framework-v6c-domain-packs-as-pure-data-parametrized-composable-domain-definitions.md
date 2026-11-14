+++
id = "P-079"
title = "Eval framework v6c: domain packs as pure data — parametrized, composable domain definitions"
status = "open"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["P-075", "P-076", "P-077"]
+++

P-075 moves domain packs to `evals/<domain>/` as TypeScript packages. Next ceiling: can a domain pack be **mostly data** — parametrized, composable, extensible without code?

## Vision

```
evals/consultation/
  domain.ts (or domain.suite-data/)   parametrized domain DEFINITION:
    schema fields      → declarative field specs (name, type, required,
                         piiCategory?, entityCategory?)
    metrics            → refs to registered metric contributions + config
    stages             → refs to registered stage contributions + config
    gatePolicies       → refs to gate presets + severity tables AS DATA
    templates          → Eta files (already data-ish)
    personas           → matrix dimensions + slot tables (already data-ish)
    locale             → LocaleProfile id (pt-BR) — profile stays code,
                         domain references it
    phiPolicy          → refs to anonymize categories + residual rules
  *.suite.ts                            manifests compose domain + caps
```

- `domain.ts` is a **typed data module** (defineDomain + Schema) — not executable pipeline code. The domain's *behavior* still lives in registered stage/metric/gate contributions (which ARE code, owned by the domain pack or capability plugins); the domain DEFINITION says which contributions + configs assemble the eval.
- Composability: domains can **extend** each other — `evals/clinical-phi/` could specialize `consultation`'s phiPolicy; `evals/phone-intake/` reuses consultation's schema with different personas/templates.
- Parametrized: matrix dimensions, template dirs, locale, entity categories become manifest/domain params — same domain code serves pt-BR consults and future es-MX variants via data.

## What stays code (honest boundary)

- Stage implementations, metric math, gate evaluation, anonymization patterns, WER — inherently code, registered as contributions.
- The domain DEFINITION (which contributions + params + templates + personas) becomes data — that's the 80% that varies between eval types.

## Acceptance

- `evals/<domain>/domain.ts` validates via Schema; contains zero executable pipeline logic.
- A domain variant (e.g. consultation-es) can be expressed as data overriding parent params — no code fork.
- eval-core unchanged (still domain-blind); apps/eval resolves domain definitions like manifests.
