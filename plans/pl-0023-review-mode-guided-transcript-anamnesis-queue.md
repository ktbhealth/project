+++
id = "PL-023"
title = "Review mode: guided transcript + anamnesis review queue (PR-review UX)"
status = "in-progress"
created = "2026-11-24"
owner = "devin"
priority = "high"
labels = ["review-queue", "product"]
links = ["P-014", "PL-022", "T-220"]
+++

## Goal

Split "view" from "review". PL-022 added the stamps and the queue
semantics; this plan turns reviewing into a dedicated, guided mode —
the physician's equivalent of a GitHub PR review: a lateral queue
with progress, two per-consultation steps (transcript evidence →
anamnesis artifact), and a finish-line summary. The end-of-day
ritual from P-014, now with a screen built for it.

## Design decisions (intent check with user)

- **Timestamps are required by default.** The pipeline currently
  discards them: `extractWhisperText` keeps only `.text`
  (`audio-consumer.ts`), while Workers AI / the OpenAI-compatible
  adapter return `verbose_json` segments
  `{start, end, text, avg_logprob}` (whisper-large-v3-turbo returns
  them by default; add `response_format: "verbose_json"` to
  `whisperInputs` only if the bound model needs it). Persist
  `transcript_segments` going forward — they power karaoke sync,
  uncertainty markers and the per-segment editor.
- **"Legacy audio" is a class, not an error.** Rows that have audio
  but no stored segments get a degraded review mode (player +
  text + confirm, no sync/incertainty markers) and are backfillable
  via the existing idempotent transcription retry path — it just
  needs to persist segments too.
- **Correction = Loro CRDT event log, never a snapshot** (user
  requirement: diffs only, always a projection). `transcript_edits`
  JSON column: `[{at, by, u: "<base64 loro update>"}, ...]`
  append-only. Projection = seed `LoroDoc` with the base
  (`corrected_transcript ?? server_transcript`), sequential
  `import` of updates, `toString()`. Multi-device edits merge
  conflict-free; undo = inverse op (append-only is native).
  Pin `loro-crdt@1.18.0` (≥7d rule; 1.19.x too fresh).
- **Server stays WASM-free**: the service validates the update
  envelope with `effect/Schema` and stores opaquely — projection
  is client-side only. The anamnesis-regenerate fn accepts the
  client-supplied projection (the physician is the trusted
  principal; audit records `editCount`, never text — PHI).
- **Base is pinned once per record** — `transcript_edits_base`
  column (`"corrected"|"server"|"transcript"`), set by the first
  `appendTranscriptEdits` when NULL. Projection base =
  `correctedTranscript ?? serverTranscript ?? transcript`
  resolved AT first-edit time (audio-less rows only carry
  `transcript`). If the LLM cleanup lands AFTER edits exist, ops
  keep projecting over the pinned base — never a silent
  misalignment; per-batch bases would need multi-doc projection
  and are rejected.
- **Stale/confirm binds on edits**: "anamnese desatualizada" =
  `max(edits.at) > record_versions[anamnesisVersion].createdAt`
  (the version row is already loaded for the integrity report —
  `consultations.anamnesis` has no timestamp of its own);
  `transcript_reviewed_edits` (op count at confirm time) mirrors
  `anamnesis_reviewed_version` — post-confirm edits surface as
  "edições novas não conferidas" — `reviewTranscript` fn updated
  to stamp it.
- **Order suggested, not enforced.** Review mode opens on the
  transcript step by default; anamnesis approval stays possible but
  shows the stale-source warning when the transcript changed after
  generation. No hard gate.
- **No bulk anything** — P-014's own caution stands; the queue is
  per-consultation guided, approve is still one explicit act.
- Chronological queue only (P-014 veto on risk/priority ranking).

## Review mode UX (PR review mapped)

- Rail: pending consultations, each a "file" with a step indicator
  (transcript unconfirmed / anamnesis unapproved / stale version /
  legacy audio), deferred rows dimmed at the bottom.
- Step 1 — transcript: audio player with segment karaoke
  (current sentence highlighted, click-to-seek), ASR uncertainty
  markers (low `avg_logprob` segments = "conflict" hunks, "next
  uncertainty" jump), 1.5–2x default speed, inline segment editor
  appending `transcript_edits` updates with struck-through original, keyboard
  map (space / arrows / C confirm / N next). Legacy rows: plain
  player + read + confirm + "legacy audio" badge.
- Step 2 — anamnesis: section click auto-highlights evidence in the
  transcript (replaces per-section "Ver evidência" toggles) and seeks
  audio when segments exist; low-confidence sections pinned first
  ("files with conflicts"); "Aprovar e próxima" is the primary
  action; stale-source warning when `transcript_edits` postdates
  generation.
- Header: "n de m revisadas" + elapsed; end of queue → session
  summary (count, time, deferred left).
- Nav: pending-count badge on the consultations nav item; detail
  page gains a "Revisar esta consulta" deep-link into the mode.

## Gaps found vs. code

- `transcribeAudio`/`extractWhisperText` drop segments — F1 must
  capture `verbose_json` (Workers AI shape; local adapter maps the
  same) and store `{start,end,text,avg_logprob}[]` per row.
- `AudioTranscriptPanel` already resolves `audioSrc` (incl. the ZK
  decrypt path) — the review player reuses it. Segments/edits
  contain PHI — under ZK they join `CONSULTATION_CIPHERED_FIELDS`
  (per-column cipher in `cipher.ts`, consumer already routes
  through it via `rowTenantIsZk`).
- `nextPendingReview` exists for sequential nav; the rail needs
  the full queue — the `list` query already returns every field
  the rail needs post-PL-022, so reuse it filtered client-side
  instead of a new `reviewQueue` query.
- Evidence→audio seek needs evidence sentences mapped to segment
  indices — substring match against segment text (same regex
  approach as `markedText`), fallback: highlight only.

## Phases

- [x] **F1 Schema + pipeline** — `transcript_segments` +
  `transcript_edits` (JSON append-only `[{at, by, u}]`) +
  `transcript_edits_base` + `transcript_reviewed_edits` (int,
  confirm binding). ZK = add `transcriptSegments` +
  `transcriptEdits` to `CONSULTATION_CIPHERED_FIELDS` (per-column
  cipher, no envelope work). Consumer persists verbose_json
  segments; backfill via existing admin retry path (T-035)
  re-transcribing legacy rows — teach it to persist segments, no
  sweep. Service `appendTranscriptEdits(id, updates[])` pins
  `transcript_edits_base` on first call (+ audit
  `consultations.transcript.edit`, ids + op count only) and fn;
  `reviewTranscript` stamps `transcript_reviewed_edits`.
  Migration + ERD. Client `projectTranscript(base, edits)` via
  `loro-crdt@1.18.0` — spike WASM load in vitest/browser first.
- [ ] **F2 Queue data + badge** — the rail reuses the existing
  `list` query (post-PL-022 it already carries id/patient/date/
  review fields) filtered client-side — NO new `reviewQueue`
  query. Pending count for the nav badge: lightweight fn. GAP:
  `PanelSidebarItem` has no badge slot — either extend kernel
  (`badge` field + host-supplied count, kernel change) or ship a
  `kind:"component"` chrome contribution rendering its own row.
  Decide with user at F2.
- [ ] **F3 Review shell** — `/consultations/review` route, queue
  rail, progress header, `?id=` deep-link, session state.
- [ ] **F4 Transcript step** — player + karaoke, uncertainty
  markers + jump, inline segment editor, keyboard map, legacy
  degraded mode + "legacy audio" badge.
- [ ] **F5 Anamnesis step** — auto-evidence on section select,
  low-confidence ordering, approve&next primary, stale-source
  warning + regenerate, transcript-confirmed step gating (suggested).
- [ ] **F6 Polish** — session summary, defer inside mode, i18n ×3,
  coverage on touched files, check, PR.

## Out of scope

Word-level karaoke (segment granularity suffices), speaker
diarization fixes, persistent sidebar widget beyond the nav badge,
bulk approve (P-014 caution), auto-regenerate on transcript edit,
converting the detail page to view-only — decide in F3 whether
inline approve buttons stay (single-item review vs strict split).
