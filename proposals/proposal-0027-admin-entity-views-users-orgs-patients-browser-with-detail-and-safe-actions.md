+++
id = "P-027"
title = "Admin entity views: users, orgs, patients browser with detail and safe actions"
status = "done"
created = "2026-11-08"
priority = "medium"
labels = ["admin"]
epic = "admin"
+++

## Idea

Entity administration views inside `/admin`: searchable/filterable
tables of users, doctors, organizations (P-022), and patients, with a
detail drawer/page per entity — status, plan, consult counts, recent
activity. Read-mostly: the actions are suspend/reactivate, reset
access, merge duplicates — not editing clinical data.

## Likely scope

- `/admin/users`, `/admin/orgs`, `/admin/patients` list views with
  search + filters
- Entity detail: profile fields, related counts (consults, members),
  timeline of admin-relevant events
- Mutation actions: suspend user/org, resend invite — each audited
  (audit-log sub-proposal)

## Why it matters

Support's #1 need: "find this user and see what state they're in."
Replaces DB spelunking during pilot onboarding and churn.

## Positioning cautions

- Read patient demographics only; clinical content stays out of admin
  unless a support case requires it (and then audited).
- Every mutation writes an audit entry — no silent admin actions.

## Depends on

- P-022 doctor registry, P-019 patient register, P-025 shell
