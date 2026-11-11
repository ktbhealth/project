+++
id = "P-005"
title = "Follow-up return reminders from recorded conduct"
status = "open"
created = "2026-11-08"
priority = "med"
labels = ["v2"]
+++

## Idea

Return reminders: "patient X should come back" based on the
recorded plan (e.g. "return in 30 days", "review the exam").

## Positioning tension

Edges into clinical decision: the system would be suggesting a
return plan. Safe version: **the physician records the return date
in the plan** (that's already documentation), and the app merely
*reminds* — a list of the day's returns, without inferring or
suggesting dates the physician didn't write.

## Likely scope

- Structured "return in" field in the anamnesis plan
- Dashboard/pending-returns list
- No NLP inferring deadlines from free text (phase 2, if any)

## Depends on

Stable patient context (patient↔consults model).
