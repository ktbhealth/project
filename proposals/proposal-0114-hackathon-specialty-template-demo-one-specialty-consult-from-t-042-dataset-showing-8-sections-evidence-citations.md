+++
id = "P-114"
title = "Hackathon: specialty-template demo — one specialty consult from T-042 dataset showing 8 sections + evidence citations"
status = "open"
created = "2026-11-24"
owner = "human:luis"
priority = "high"
labels = ["demo", "hackathon"]
links = ["T-042", "T-091", "P-065"]
epic = "mvp"
milestone = "hackathon"
due = "2026-11-30"
estimate = 200
+++

## Context

P-065 wedge is "specialty-first, locale-native clinical scribe" but the specialty.* / locale.* pack model (T-091) is unbuilt architecture. For the hackathon MVP we don't need the pack system — we need to *show* specialty-aware documentation.

## Proposal

Pick ONE specialty (cardiology suggested — richest vocabulary, clearest demo) and produce:

- A specialty consult transcript from the T-042 synthetic dataset (or recorded live)
- The generated anamnesis: 8 sections with evidence citations
- A short demo narrative: "the scribe knows cardiology vocabulary — PT-BR, CID-10, TUSS"

Evidence citations are the differentiator vs. generic scribes: every section shows the verbatim transcript sentences that ground it — the "never invents findings" guarantee made visible.

## Out of scope

- specialty.* plugin pack architecture (T-091 stays post-hackathon)
- Locale packs beyond pt-BR
- Custom per-specialty section schemas
