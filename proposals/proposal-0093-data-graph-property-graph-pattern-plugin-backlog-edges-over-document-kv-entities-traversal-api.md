+++
id = "P-093"
title = "data.graph: property-graph pattern plugin (BACKLOG — edges over document/kv entities, traversal API)"
status = "deferred"
created = "2026-11-17"
owner = "devin"
priority = "low"
links = ["P-006", "P-089", "P-090"]
+++

## Problem

Relationship-heavy queries (patient → clinic → doctor → consult → documents → referrals) are the known weak spot of both relational schemas (deep self-joins) and document stores (no traversal). Today nothing needs graph queries badly enough to justify an engine — this proposal exists to record the deferred option with explicit revisit triggers so the decision isn't relitigated.

## Proposal (DEFERRED — backlog)

`plugins/data-graph` (`@ktb/data-graph`): `data.graph` pattern plugin — property-graph layer over the existing store.

- Nodes = references into documents/kv/drizzle entities (no data duplication — graph stores edges + properties only).
- Edges typed (`consults_for`, `member_of`, `referred_by`, `consent_scope`), direction, properties, valid-time optional.
- Service: `link/unlink`, `neighbors`, `paths` (bounded BFS/DFS with depth+fanout caps), `ego` subgraph export.
- Storage: `graph_edges` table (src_kind+src_id, edge_type, dst_kind+dst_id, props JSON, tenant scope) + indexes; recursive CTEs for traversal where SQLite supports them, JS fallback for complex patterns.
- Tenant fencing mandatory on every traversal.

## Explicit non-goals / revisit triggers

- DO NOT BUILD until a real query pattern needs multi-hop traversal: candidates P-006 (consent-scoped history sharing across clinics), P-022 (multi-practitioner referral networks), P-081 (portal audience graphs).
- document-store `ref` fields + bounded traversal API cover depth-1..2 needs meanwhile.
- If built, it must reuse data.documents for node payloads — no parallel entity store.

## Relations

- Family: P-089 (documents — node payloads), P-090 (kv — not graphed).
- Status: BACKLOG by design — cheapest correct option is refs+traversal until pull is proven.
