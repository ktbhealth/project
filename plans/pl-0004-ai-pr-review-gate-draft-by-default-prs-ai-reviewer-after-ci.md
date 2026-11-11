+++
id = "PL-004"
title = "AI PR review gate: draft-by-default PRs + AI reviewer after CI"
status = "open"
created = "2026-11-09"
owner = "human:emanuel"
priority = "medium"
labels = ["ai-review", "github-actions"]
+++

# Plan: AI PR review gate — draft-by-default PRs + AI reviewer after CI

## Context
CI is being rolled out across the org (T-030, PL-002) and the product repo now enforces a
100% coverage gate (T-073, T-074, T-049, T-050). The next step is to put a technical reviewer at the end
of the pipeline: PRs open as **draft**, CI runs, and when CI is green an **AI agent reviews
the diff and approves or rejects it with technical grounds** (cited file/line, rule
violated, severity). A human still merges.

Goals
- Every PR gets a consistent, evidence-based first review within minutes of CI passing.
- PRs are draft until they clear CI + the AI review (reduces review noise for humans).
- One implementation reused by all org repos, versioned, rubric kept in one place.

Non-goals (for now)
- Replacing human review or auto-merging.
- Reviewing PRs from forks (see Security).
- Orchestration via Peregrin — GHA first (T-026 can wrap the same job later).

## Flow
```
open PR (draft) -> ci (check/test/build) -> ai-review job (only if ci green)
   -> verdict JSON -> post review + set `ai-review` status
        approve          -> review APPROVE, label ai-review:approved, mark ready for review
        request_changes  -> review REQUEST_CHANGES + inline comments, stays draft
        comment          -> advisory comment only (uncertain / needs human)
new push -> older run cancelled -> CI + AI review run again
```

## Design

### 1. Draft by default
To my knowledge GitHub has no repo/org setting to open PRs as draft by default — verify in
the spike. Mechanisms, in order of preference:
1. Convention + tooling: `gh pr create --draft` in our agent/CLI workflows; add the rule to
   the root `AGENTS.md` (and a PR template reminder).
2. Safety net: a small workflow on `pull_request: opened` that converts non-draft PRs to
   draft (GraphQL `convertPullRequestToDraft`, needs `pull-requests: write`). Skip for bots
   (dependabot) and for PRs with label `ready-no-ai`.
The AI job marks the PR "ready for review" (`gh pr ready`) only on APPROVE.

### 2. Where it runs
- Reusable workflow `ai-review.yml` in `ktbhealth/.github` (`on: workflow_call`), tagged
  `v1`; every repo calls it as a final job of its `ci.yml`:
  `ai-review: needs: ci`, guarded by
  `if: github.event.pull_request.head.repo.full_name == github.repository`.
- Prefer `needs:` inside the same workflow over `workflow_run`: no cross-workflow trigger
  and no "pwn request" surface. Never use `pull_request_target` + checkout of PR head.

### 3. The reviewer
- `anthropics/claude-code-action@v1` (pin to a release tag/SHA; the repo is active).
  Open question for the spike: structured output. If the action cannot enforce a JSON
  schema, have the agent emit one fenced JSON block and validate it in a follow-up step.
- Tools: read-only only (Read, Grep, Glob, `gh pr diff/view`, `gh api` GET). No write Bash.
- Input: PR title/body, diff, changed files, CI results, the repo's `AGENTS.md`, the shared
  rubric. Output schema:
  `{ verdict: approve|request_changes|comment, summary, findings: [{file, line, severity,
  rule, rationale, suggestion?}] }`.
- Evidence rule: a finding must cite file:line from the diff and the rule it violates.
  `request_changes` needs at least one `blocker`; when unsure the verdict is `comment`,
  never `request_changes`.

### 4. Rubric (shared, in `ktbhealth/.github/ai-review/rubric.md`)
- Correctness and edge cases; error handling at boundaries.
- Tests: new behavior covered, no `v8 ignore`/excluded files, coverage thresholds not lowered.
- Org rules from `AGENTS.md`: English everywhere; `ui` is the single source of truth (no
  vendored components, bump the git-tag dep); no edits to `node_modules` or generated files
  (paraglide, `routeTree.gen.ts`, drizzle migrations, `dist/`, `coverage/`).
- Security/privacy: this is health data (LGPD) — secrets, authz checks, PII in logs,
  injection, unsafe deserialization.
- Migrations and breaking changes (drizzle, `@ktbhealth/ui` consumers, public APIs).
- Dependencies: new/changed deps, lockfile consistency, postinstall trust.
- Scope: unrelated changes, missing myco task reference (T-NNN) in the PR body.

### 5. Outcome handling and branch protection
- The AI result is a **required status check** `ai-review`, NOT the approval that unlocks
  merge: branch protection (T-054/T-057) requires `ci` + `ai-review` + one human approval,
  with CODEOWNERS on `vitest.config.ts` and the workflow files.
- Labels: `ai-review:approved` / `ai-review:changes-requested` / `ai-review:skip`
  (maintainer override, audited in the run log).

### 6. Identity, secrets, permissions
- Job permissions: `contents: read`, `pull-requests: write`, `statuses: write`, nothing else.
- `ANTHROPIC_API_KEY` as an org secret scoped to the repos that use it.
- Decision: GITHUB_TOKEN (bot `github-actions`, requires org setting "Allow GitHub Actions
  to create and approve pull requests") vs a dedicated GitHub App `ktb-reviewer` with its
  own identity. Recommendation: GitHub App, so approvals/comments have a stable author and
  can be audited and revoked independently.

### 7. Security (all org repos are PUBLIC)
- PR title/body/diff/comments are untrusted input -> prompt injection. Mitigations:
  same-repo branches only (forks skipped, secrets unavailable anyway), read-only tools, no
  secrets in the agent env beyond the API key, never execute PR code in this job, validate
  the model output against the schema before acting, cap diff size (skip + comment "too
  large for AI review" above a threshold).
- Treat the agent's text as data: never pass it to a shell or `eval`.

### 8. Cost and noise control
- Runs only after CI is green, one run per head SHA, `concurrency` cancels superseded runs.
- Skip docs-only / lockfile-only / bot PRs; `ai-review:skip` label.
- Budget guard: monthly spend alert on the API key; log tokens per run in the job summary.

### 9. Peregrin (T-026, later)
Keep the GHA job a thin wrapper around "run reviewer with inputs X" so Peregrin can
orchestrate the same step as a campaign without changing the rubric or the schema.

## Dependencies
- T-030 CI PRs merged (product #3, storybook #2, website-draft #1, slidev-theme-ktb #1,
  presentation-draft #1, ui #2); product coverage gate green (PRs #5, #6).
- T-054 / T-057: branch protection + CODEOWNERS (required checks).
- T-026: Peregrin as CI orchestrator (later integration).

## Phases
- [ ] Phase 0 — Spike on one low-risk repo (`storybook`), advisory only: verify draft-by-
      default options, action version + structured output, `gh pr ready`, permissions.
- [ ] Phase 0b — Calibrate: replay the reviewer on merged PRs (ui#1, storybook#1,
      product#1-#6) and on a seeded bad PR (coverage dropped, vendored component, edited
      generated file); tune the rubric until findings are accurate and non-noisy.
- [ ] Phase 1 — Reusable workflow in `ktbhealth/.github` (v1) + rubric; enable advisory
      (comment only) on all repos.
- [ ] Phase 2 — Make `ai-review` a required status check; GitHub App identity; CODEOWNERS.
- [ ] Phase 3 — Draft-by-default enforcement workflow + auto "ready for review" on approve.
- [ ] Phase 4 — Metrics (false positives/negatives, cost per PR) and Peregrin hand-off.

## Acceptance criteria
- A PR opened by us starts as draft; after CI is green it receives an AI review with cited
  evidence within ~5 minutes.
- Approve => PR marked ready + `ai-review` check green; reject => inline comments + red
  check + stays draft; uncertain => advisory comment.
- Fork PRs never reach the agent or secrets.
- Merge still needs a human approval; the AI cannot merge or bypass protection.
- Rubric and workflow live in one place (`ktbhealth/.github`) and are versioned by tag.

## Open questions
- Which model/effort per review, and the per-PR / monthly budget?
- GitHub App vs GITHUB_TOKEN (recommendation above: App).
- Should AI approval auto-mark ready, or only comment and leave the human to mark ready?
- Required check from day one of Phase 2, or after N weeks of calibration data?
- Where do per-repo extra rules live — only in each repo's `AGENTS.md`, or an optional
  `.github/ai-review.md` override?
