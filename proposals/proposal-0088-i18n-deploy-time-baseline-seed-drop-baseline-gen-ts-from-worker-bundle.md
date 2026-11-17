+++
id = "P-088"
title = "i18n: deploy-time baseline seed — drop baseline.gen.ts from worker bundle"
status = "in-progress"
created = "2026-11-17"
priority = "med"
links = ["P-083"]
+++

Context: PR #77 shipped baseline.gen.ts (~4k rows) inside the worker bundle so I18nRuntime could auto-seed empty DBs. Works, but every worker pays ~300-400KB raw for data only needed once.

Decision (agreed with owner):
- DB stays canonical; plugin i18n.ts keeps typed interfaces + per-locale catalogs as seed source + fallback.
- baseline.gen.ts remains generated, but is consumed ONLY by seed tooling — not statically imported by runtime.ts.
- Auto-seed becomes lazy (dynamic import only when namespaces table is empty) or dev-gated; prod bootstrap = deploy pipeline runs `bun run i18n:import-baseline --sql | wrangler d1 execute --remote` (tool already exists, idempotent).
- NO bidirectional DB->code sync: code->DB only at bootstrap; DB->file only via panel export / --sql dump (already built). Runtime edits never auto-commit to git.
- Add `i18n:doctor` check: reports namespace/locale/message counts, missing namespaces vs manifest, draft/published split, empty-DB detection — for post-deploy verification.

Open detail: lazy dynamic-import chunk still ships bytes to the worker (loaded only when DB empty). Full exclusion would need build-time stripping — acceptable residual, revisit if bundle budget tightens.
