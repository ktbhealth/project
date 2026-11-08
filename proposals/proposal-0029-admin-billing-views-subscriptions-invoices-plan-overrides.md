+++
id = "P-029"
title = "Admin billing views: subscriptions, invoices, plan overrides"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["admin"]
epic = "admin"
+++

## Idea

Billing administration in `/admin`: subscription state per org/doctor
(P-024), invoice list, plan overrides (comp, extended trial), and
links into the Stripe dashboard for deep actions. Read + a small set
of safe mutations; money-moving actions stay in Stripe's portal.

## Likely scope

- `/admin/billing`: org/doctor → plan, status, renewal date, MRR-ish
  rollup
- Invoice list filtered by org/status
- Plan override action (grant/revoke comp plan) — audited
- Deep links to Stripe dashboard objects for refunds etc.

## Why it matters

When billing exists, "why didn't this clinic get charged" / "comp this
pilot customer" are daily ops questions. Keep the admin thin — Stripe
is the system of record for money.

## Positioning cautions

- Never store card data or mark invoices paid manually — Stripe owns
  that truth.
- Overrides are plan-state changes, not fake invoices.

## Depends on

- P-024 Stripe billing, P-022 orgs, P-025 shell
