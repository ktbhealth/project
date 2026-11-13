+++
id = "P-012"
title = "Dot phrases: reusable text snippets for anamnesis and plan"
status = "open"
created = "2026-11-08"
priority = "low"
labels = ["platform-qol", "product"]
+++

## Idea

Personal shortcuts that expand into longer clinical text while
the physician reviews/edits the draft: ".ret30" → "Retornar em
30 dias para reavaliação", ".exnormal" → standard exam paragraph.

## Likely scope

- Personal snippet library: trigger → expansion text
- Expansion inside anamnesis fields and plan during review
- Starter pack of common PT-BR phrases per specialty
- Sync across devices (physician's account)

## Why it matters

Review/approve is where the time goes after transcription; dot
phrases compress the repetitive 60% of editing. Cheap to build,
felt daily.

## Positioning cautions

- Snippets insert *text the physician chose* — they don't
  generate clinical content. No suggested phrases derived from
  the transcript (that's decision territory).
- Starter phrases must be specialty-neutral or clearly labeled
  per specialty.

## Depends on

Review UI; nothing structural.
