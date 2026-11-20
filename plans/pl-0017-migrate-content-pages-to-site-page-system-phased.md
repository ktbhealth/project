+++
id = "PL-017"
title = "Migrate content pages to site page system (phased)"
status = "claimed"
created = "2026-11-20"
owner = "human:Luis-Emidio"
priority = "med"
links = ["P-107", "P-109", "P-112", "PL-014"]
epic = "site-builder"
+++

# Plan: page migration (P-112) — strictly phased

## Phase A — App/marketing content pages (first)
- Audit the ~9 app-context `pages` contributions + `landing:` kinds: classify content vs functional.
- Migrate content pages to `pages` rows (template-instantiated via P-107 or hand-built builderJson).
- Keep old paths stable via P-106 route keys; deprecate migrated descriptors.

## Phase B — Platform content surfaces (selective)
- Audit `/platform/*`: migrate only true content pages; admin consoles (audit, i18n, kv, rbac, document-store, eval, etc.) STAY CODE — they are tools, not documents.
- Dashboard widgets evaluated case-by-case.

## Phase C — Domain pages (LAST — team actively working)
- Wait for domain work to settle; migrate only informational/fragment surfaces.
- Core consult/patient/agenda UIs remain code — workflows, not content.
- Domain blocks stay in their plugins as element kinds (PatientCard etc.).

## Cross-cutting
- Every migration: template snapshot first (rollback = re-point descriptor), route key stable, delete-hook cleanup registered.
- Success metric per phase: page renders identically, editable in builder, old descriptor removed.
