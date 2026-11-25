+++
id = "P-118"
title = "Shared transcript pipeline + DecisionsPort (System One/OpenJev)"
status = "open"
created = "2026-11-25"
priority = "high"
+++

# Shared transcript pipeline + DecisionsPort (System One/OpenJev)

## Problem

The eval pipeline got the real upgrades (reversible `EditList` cleanup,
ASR error-range gating via a decision-model judge, openai-compat
corrector), but the app's `processAudioJob` still runs the old path:
Workers AI Whisper → free-text rewrite (`LlmPort.generate` /
`LanguageModel.generateText`). No edits, no gate, segments discarded.
Decision models exist only inside eval (`EvalPort` + `makeSystemOneEval`
in node-bound `eval-adapter-local`); admins cannot pick one like an LLM
endpoint.

## Proposal

One composable, plugin-configured pipeline — `ASR → gate → correct →
applyEdits` — consumed verbatim by `eval run` and the app job. Stage
functions already live in `@ktb/eval-consultation` (`pipeline.ts`,
`asr-ranges.ts`) as pure port orchestration; this wires both sides to
them.

### 1. `DecisionsPort` (`@ktb/eval-core`)

Rename/generalize `EvalPort.judge` → `DecisionsPort.decide(state,
questions) → answers`. It is a decision-model port, not eval-specific.
Keep `EvalPort` as a deprecated alias; stage id `judge` unchanged.

### 2. `@ktb/ai-adapter-systemone` (new package)

Move the `/v1/systemone` wire client out of `eval-adapter-local` into a
fetch-only, workerd-safe adapter (same pattern as `ai-adapter-openai`):
`makeSystemOneDecisions({baseUrl, model?, apiKey?, timeoutMs}) →
Layer<DecisionsPort>`. One wire covers OpenJev/`wev serve`,
razorback16/openjev, local-jev, hosted TypeSafe. `eval-adapter-local`
re-exports; `eval-cli` `systemone` key and ai-provider share the code.

### 3. ai-provider plugin — decision endpoints

- `ai_endpoint.kind` += `"systemone"` (reuses base_url/model/secret_ref).
- New `DecisionsFactory` port + `PlatformDecisions` capability mirroring
  `PlatformLlm`: per-call resolve of enabled `scope='platform'`
  `is_default` systemone endpoint → `Layer<DecisionsPort>`; null →
  fallback.
- Admin endpoint form: "System One (OpenJev/TypeSafe)" kind + presets.

### 4. `plugins/transcript-pipeline` (new plugin)

Contracts stay in `@ktb/eval-consultation`; the plugin contributes the
composition:

- `TranscriptPipeline` service: `run({audio|text, segments?, taskId,
  stages}) → {asr?, corrected?, edits, records, gateStats?}`.
  `serviceOption`s each port (`AsrPort`, `LlmPort`, `CleanupGate`,
  `DecisionsPort`) and runs the existing stage functions; absent port =
  stage skipped, recorded honestly.
- `CleanupGate` impl consuming the **ambient** `DecisionsPort` (today's
  `makeSystemOneCleanupGate` builds its own EvalPort) — the admin's
  chosen endpoint powers prod gating. Keep the self-contained env
  variant for the CLI.
- Config: eval keeps manifest `stages` + `EVAL_CLEANUP_GATE`; app gets
  `TRANSCRIPT_GATE` / `TRANSCRIPT_GATE_THRESHOLD` env fallbacks with
  platform-endpoint precedence.

### 5. Rewire consumers

- `processAudioJob`: pipeline service replaces `correctTranscript`
  free-text; `correctedTranscript`/`enhancementEngine` unchanged; audit
  detail gains `edits: n` / `gate: name`.
- App ASR: Workers-AI-binding `AsrPort` adapter returning `AsrResult`
  with `segments` (verify binding returns them; else sentence-level
  pseudo-segments — `charRangesFor` maps either way).
- Env fallback (`LlmConfig`): wrap `LanguageModel` providers as
  `LlmPort.correct` in `anamnesis-llm` so env path also emits edits.
- `makeEvalRunner` (in-app validator): `cleanupTranscript` → gated
  variant when `CleanupGate` bound.
- `eval run`: unchanged; same functions, same ports.

## Config matrix

| concern | eval | app |
|---|---|---|
| stages | suite manifest | fixed asr→gate?→cleanup |
| ASR | suite adapter | Workers AI binding |
| corrector | `llm` capability | platform endpoint → env |
| gate | `EVAL_CLEANUP_GATE` | platform decision endpoint → env |
| judge | `eval` capability | n/a |

## Risks

- Workers AI binding may return text only → pseudo-segments for gate.
- OpenJev probabilities compress (~0.3–0.55); threshold 0.5 default,
  tunable + logged per job.
- `eval-adapter-local` refactor must keep CLI behavior identical.
