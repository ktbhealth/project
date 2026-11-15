+++
id = "P-081"
title = "Eval validator platform: persona-driven generation + in-app approval/metrics UI"
status = "open"
created = "2026-11-15"
owner = "devin"
priority = "high"
+++

# Eval validator platform — personas, workflow, in-app metrics

## Problem

The eval pipeline measures WER correctly but the *inputs* are not
clinically believable and the *outputs* are not reviewable inside the
product:

- Patient/doctor are not personas — no gender, age, social class,
  education; register is only "formal/colloquial". Result: a colloquial
  patient says "cefaleia" (technical term a lay person rarely uses)
  — s-12bc075c.
- One voice per sample, not per speaker — and voice is not tied to
  persona gender.
- Review happens in a throwaway HTML file, outside the app, with no
  approval workflow, no progress tracking, no audit of who approved
  what.

## Design

### 1. Rich personas (domain, evals/consultation)

`Persona` gains: `gender` (m/f/nb), `age` (years, not just band),
`socialClass` (a/b/c/d/e — IBGE-style), `education`, `name` (synthetic
tag), `speakerRole` (patient|doctor). A **cohort** becomes pairs:
`{ patient: Persona, doctor: Persona, region, specialty }` — the
doctor is also generated (gender, register, specialty-matched).

Script generation becomes persona-aware:
- **Lexical register** driven by education/class: colloquial patients
  say "dor de cabeça", not "cefaleia"; formal/educated patients may
  use technical terms. Each content pool carries register variants —
  already partially done, extend to *vocabulary* not just tone.
- **Speech style** driven by age (elderly = more repetition, shorter
  sentences) and region (existing markers).
- **Voice** resolved from `speaker.gender + region` via the voice map —
  `speakerVoices` moves from static config to per-sample derivation.

### 2. Approval workflow (new plugin `eval.validator`)

Registry plugin following the existing plugin model (patients.services
→ consultations pattern). State machine per persona-batch:

```
generated → under_review → approved | rejected(with feedback)
approved → eval_queued → running → done | failed
```

- Feedback is free-text + per-field flag (voice/register/script).
- Rejected personas go back to generation with the feedback appended
  as constraints — auditable loop.
- Auth: new role `eval.validator` in the RBAC vocabulary — separate
  login/scope from clinical users. Validators see only the eval UI.

### 3. In-app UI (plugin pages, design system from @ktb/ui)

Route under `/platform/eval`:
- **Persona gallery**: cards per generated pair — patient + doctor
  avatar-less chips, gender/age/class/education/region/specialty,
  expandable script with per-turn speaker coloring, audio preview
  (TTS sample), approve/reject buttons + feedback input.
- **Eval dashboard**: per-run progress bar (n/total, %), live stage
  counts (transcribe/cleanup/judge), elapsed time.
- **Results view**: per-sample WER/CER, word-diff inline (the
  Levenshtein alignment from inspect_gen.py ported to a component),
  judge verdicts + confidence bands, gate result, per-persona
  breakdown (WER by region/class/register — the demographic slicing
  the threshold needs).

### 4. Progress reporting (backend)

`EvalRun` gains a `progress` field updated per-sample (the store
already counts samples — add a `run_progress` row or compute from
`results` count vs manifest `sampleCount`). Server fn returns
`{ done, total, stage, etaMs }` — the dashboard polls.

## Tasks

- T: Persona model — gender/age/class/education/name + doctor personas
- T: Persona-aware scriptgen — vocabulary per education/class, age
  speech style, lay-vs-technical term mapping
- T: Voice resolution from persona gender+region; per-speaker voices
- T: `eval.validator` plugin — schema (persona batches, feedback,
  run progress), ports for store
- T: RBAC role `eval.validator` + separate login/scope
- T: Persona gallery UI (approve/reject + feedback)
- T: Eval run trigger + progress polling UI
- T: Results view — diff, metrics, judge verdicts, demographic slices
- T: Port inspect_gen.py diff → shared component (domain renders ops,
  UI renders spans)

## Non-goals

- Real PHI — personas remain synthetic tags.
- Changing the ASR/cleanup/judge pipeline itself — this is the
  *validation surface*, not the pipeline.
