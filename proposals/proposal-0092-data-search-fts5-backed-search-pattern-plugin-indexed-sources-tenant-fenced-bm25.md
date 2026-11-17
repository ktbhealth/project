+++
id = "P-092"
title = "data.search: FTS5-backed search pattern plugin (indexed sources, tenant-fenced, bm25)"
status = "open"
created = "2026-11-17"
owner = "devin"
priority = "low"
links = ["P-021", "P-089", "P-090"]
+++

## Problem

Full-text and faceted lookup across plugin data outgrows `LIKE`/JS filtering: patient directory search (P-021), consult transcript search, document-store collection queries at scale. document-store's `$regex` is unindexed and ReDoS-prone — fine for small collections, wrong for directory-scale search.

## Proposal

`plugins/data-search` (`@ktb/data-search`): `data.search` pattern plugin — an FTS adapter over SQLite FTS5 (already in the D1/SQLite plane), NOT a new engine.

- Effect service `Search`: `index(collection|source, id, fields)`, `remove(source, id)`, `query(q, { filters?, limit?, offset? })` → ranked ids; callers hydrate docs via their own store.
- Sources declared as data: which fields are indexed, tokenizers (unicode61 default; trigram for substring/CJK needs — pt-BR names OK with unicode61), language hints.
- Storage: FTS5 virtual tables created per source at registration (`CREATE VIRTUAL TABLE … USING fts5`) — DDL is fixed-shape, not per-plugin schema churn; one `search_sources` registry table for metadata.
- Integration path: document-store collections opt in via a `search` field on the collection def → auto-index on write (keeps P-089 ergonomic); drizzle entities can index selected columns explicitly.
- Tenant fencing: scope column in FTS table + mandatory scope filter on query.
- Ranking: bm25 default; snippet/highlight optional.

## Non-goals

- Not a search engine cluster, no vector/semantic search (eval/embedding retrieval is a different port — see llm-eval plugins).
- Build trigger: patient directory (P-021) or consult search hitting scale/pain; trigram tokenizer covers pt-BR substring needs.

## Relations

- Family: P-089 (documents — primary index source), P-090 (kv — not searchable by design).
- Feeds: P-021 (patient directory), P-010 (day dashboard prep context), P-006 (consent-scoped history sharing search).
