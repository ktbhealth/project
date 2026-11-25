+++
id = "P-014"
title = "Review queue: end-of-day batch approval of consult drafts"
status = "open"
created = "2026-11-08"
priority = "low"
labels = ["platform-qol", "product"]
+++

## Idea

An inbox-style queue of consult drafts awaiting physician
review, with batch actions: approve, approve-and-export,
defer to tomorrow. Turns review from per-consult context-switch
into a focused end-of-day ritual.

## Likely scope

- Queue listing drafts by consult/date/patient, oldest first
- Per-item diff-ready view (what the AI drafted vs. final)
- Bulk approve for low-risk fields? — NO, see cautions
- "Finish tomorrow" parking; queue survives across days

## Why it matters

Documenting after hours is the pain the product exists to kill;
a queue makes the remaining human step a 10-minute ritual
instead of scattered interruptions. Directly supports the
human-in-the-loop promise without weakening it.

## Positioning cautions

- Batch actions can approve *whole drafts* but each consult
  still requires an explicit per-consult approval — no
  "approve all" that signs unreviewed records.
- Queue ordering is chronological only; no risk/priority
  ranking of patients.

## Depends on

Consult drafts pipeline; agenda optional (queue works without
P-008, agenda just feeds it).
