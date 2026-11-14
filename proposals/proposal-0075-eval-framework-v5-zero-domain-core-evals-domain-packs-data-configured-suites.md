+++
id = "P-075"
title = "Eval framework v5: zero-domain core — evals/<domain>/ packs + data-configured suites"
status = "open"
created = "2026-11-14"
priority = "med"
labels = ["architecture-platform"]
links = ["T-042", "T-118", "P-070", "P-074"]
+++

Evolution of P-074 (implemented in PR #26). Pushes the boundary one level further: the framework must have **zero domain logic or knowledge** — not just "no transcript knowledge" but no eval-domain knowledge at all. The first eval (consultation) becomes one data-configured suite; everything consult-specific lives under `evals/`.

## Problem with current shape

PR #26 put the domain pack in `plugins/eval-consult` — a plugin registry citizen for code that is eval-internal. That couples eval domain code to the product plugin system (we already had to exclude eval plugins from the apps/web glob). The honest boundary: **plugins/ = shared capability declarations; evals/ = everything eval**.

## Proposed shape

```
packages/eval-core/            PURE framework — zero eval-domain knowledge:
                               suite manifest Schema, ContributionRegistry,
                               runBatch, RunRecord/report/diff, gate ENGINE
                               (policy-agnostic), generic ports.
                               Knows: stages, metrics, gates, caps are
                               registered contribution ids. Nothing else.

plugins/                       capability registrars ONLY (unchanged):
                               transcription.tts/asr, llm.eval, storage.dataset
                               — declare ports/adapters/stages they provide.

packages/eval-adapter-*/       adapter implementations (exempt edge).

evals/
  consultation/                DOMAIN PACK — everything consult-specific:
    package.json tsconfig      (workspace package @ktb/eval-consultation)
    src/schema.ts              Sample, gold entities, ClinicalCategory
    src/pipeline.ts            stage fns (transcribe/cleanup/judge impls)
    src/metrics.ts             WER/CER/entity-verdicts/risk-score
    src/gates.ts               auto-accept policy + consult.strict severity
    src/personas.ts scriptgen.ts generate.ts
    src/locale.ts locales/pt-BR.ts anonymize.ts acp.ts phi.ts
    templates/pt-BR/           Eta templates
    src/index.ts               consultContributions (metric/gate/stage refs)
  triage/                      future domain packs — same shape
  *.suite.ts                   suite manifests — pure data (defineSuite)
```

- `plugins/eval-consult` dissolves → `evals/consultation/`. Plugin manifests for domain packs are unnecessary — domains aren't consumed by apps/web, only by apps/eval.
- Suite manifests move inside their domain dir or stay at evals/ root — pick one convention (`evals/<domain>/<name>.suite.ts` is cleanest: suite colocated with its domain).
- apps/eval discovers `evals/*/` domain packs + `evals/**/*.suite.ts` manifests via explicit imports (Bun/Node — no glob transform needed if we keep an index or fs-scan).
- eval-core's ContributionRegistry remains the mini-registry until P-070's generalized kinds land; then eval contribution kinds (`eval.stage`, `eval.metric`, `eval.gate`, `eval.capability`, `eval.domain`) register there.

## Why

- Framework literally cannot accumulate domain logic — there's no plugins/eval-consult to tempt it.
- evals/ becomes the single, browsable home for everything eval: domains, suites, manifests.
- Adding an eval type = `evals/<domain>/` + `evals/<domain>/<suite>.suite.ts`; zero framework/plugin changes.
- Domain packs can still be versioned/reviewed as code; manifests stay pure data.

## Migration

1. `git mv plugins/eval-consult → evals/consultation` (+ package.json/tsconfig/vitest adjustments; workspace glob already covers? check).
2. Update importers: capability plugins (`@ktb/eval-consult/ports` → `@ktb/eval-consultation/...`), apps/eval, evals/ manifests.
3. Remove eval-consult from app-host exclusion list (it's gone); keep capability plugins excluded until P-070 lands (they're still plugins/ citizens — actually with P-070 they can join properly).
4. `evals/consultation/consult-asr.suite.ts` (or keep evals/consult-asr.suite.ts — decide convention, document).

## Acceptance

- eval-core contains zero domain identifiers (grep: no consult/transcript/pt-BR/clinical strings).
- `evals/consultation/` is self-contained; a second domain pack can be added by copying the shape.
- All tests green; CLI works identically.
