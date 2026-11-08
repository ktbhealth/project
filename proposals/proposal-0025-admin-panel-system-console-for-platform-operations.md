+++
id = "P-025"
title = "Admin panel: system console for platform operations"
status = "done"
created = "2026-11-08"
priority = "high"
labels = ["admin"]
epic = "admin"
+++

## Idea

A system-alike admin panel: an internal back-office console at `/admin`
for operating the KTB Health platform — not a doctor-facing feature.
Dense, utilitarian UI (system console aesthetic, not marketing polish):
sidebar nav, tables, status badges, raw-ish data with sane formatting.

Umbrella for the admin surface; each capability is its own proposal:

- shell + authorization (who can even see `/admin`)
- entity administration (users, doctors, orgs, patients)
- ops health (queues, transcription jobs, webhooks, storage)
- billing administration (subscriptions, invoices, overrides)
- audit log (every admin mutation recorded)

## Why it matters

Before real users, every operational task is a database query or a
manual Cloudflare dashboard visit. The moment there's a pilot, "why is
this transcription stuck" or "disable this account" needs a UI. The
admin panel is also where support/ops headcount scales without code
deploys.

## Positioning cautions

- `/admin` is internal-only. Authz must be a hard gate (role check
  server-side, not hidden nav). Never expose patient clinical content
  beyond what's needed for support.
- System-alike means information-dense and honest — no vanity metrics.
- Patient data shown in admin views is still LGPD-regulated; access
  itself should be audited (see audit-log sub-proposal).
- Build thin: admin reads the same services/APIs the product uses;
  don't fork business logic for the console.

## Depends on

- P-022 doctor registry (orgs/members are what admins manage)
- P-024 Stripe billing (billing admin surfaces subscription state)
- authz package for the admin role
