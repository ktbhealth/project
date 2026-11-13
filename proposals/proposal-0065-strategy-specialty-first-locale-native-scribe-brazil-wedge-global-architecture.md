+++
id = "P-065"
title = "Strategy: specialty-first locale-native scribe — Brazil wedge, global architecture"
status = "in-progress"
created = "2026-11-12"
owner = "human:luis"
priority = "high"
labels = ["business", "strategy", "wedge"]
+++

## Context

Post-`v0.1.0` snapshot (2026-11-12). Hackathon draft submitted; next round in ~1 month. Board has 19 open tasks and 35 open proposals — polish needs a filter. `business_plan.md` lists 4 differentiation options ("pick 1–2") and defers MVP specialty ("whichever we get access to"). This proposal locks the strategy layer so all downstream work (UX polish, feature triage, CI) has a single filter.

## Decisions (2026-11-12, with luis)

- **Market**: starts in Brazil, must scale globally from day one.
- **Network**: global and local — pilot recruitment is not geography-blocked.
- **Hackathon goal**: both technical excellence AND visible traction.

## Wedge

**"The specialty-first, locale-native clinical scribe."**

Brazil is locale #1, not the identity. PT-BR/CID-10/WhatsApp reality is the go-to-market wedge; the product architecture must treat locale and specialty as parameters, not hardcodes.

Rejected as lead wedge: LGPD-native (table stakes, phase 2), EMR integration (clipboard export suffices for v1).

## Global-scalability invariants

Every Brazil-specific concern must be a parameter:

- **Anamnesis structure** → specialty packs × locale packs (CID-10 = BR/WHO variant; US = ICD-10-CM).
- **Transcription** → port already locale-agnostic (P-001, ADR-0001); PT-BR is first adapter profile.
- **Channel** → channel-agnostic per P-001; WhatsApp/dictation is BR GTM, not product identity.
- **Compliance** → LGPD now; design must not preclude GDPR/HIPAA-shaped regimes later.
- **Plugin model** → `specialty.*` / `locale.*` packs are a coherent extension of `domain.concern` plugin ids (see P-031–P-064 architecture workstream).

## Beachhead

**Psychiatry** (default pending luis confirmation): long consults, heavy documentation burden, template-rich anamnesis, high willingness to pay for time savings. Revisit if pilot recruitment surfaces a stronger-access specialty — the wedge does not depend on it, the template does.

## Pilot plan

- Recruit 5–10 physicians (global + local network; hackathon as channel).
- 2-week free pilot; success = continued usage + stated willingness to pay ~R$99–149/mo.
- Instrument: which competitor they tried, whether price or price+language was the blocker (open question from business_plan.md).
- Pilot signup is the CTA on website/deck/docs — every artifact funnels to it.

## Demo script outline (3 min)

1. Landing → positioning line ("documentation-only; physician decides").
2. Doctor starts consult → consent → record/transcribe.
3. Structured PT-BR anamnesis appears → doctor edits/approves.
4. Export to EMR (clipboard) → done.

Golden path = consent → record → transcribe → review → approve → export. All polish work filters through "does this make the demo better?"

## Work items (rough)

- Confirm beachhead specialty with luis (psychiatry default).
- Write full pilot recruitment plan (channels, list of 10 target doctors, tracking).
- Align website/deck/docs copy to wedge sentence (extends T-082).
- File `specialty.*`/`locale.*` pack proposal against plugin architecture.
- Define demo environment: seeded patient + synthetic consult (ties to T-042 dataset).

## Open questions

- Beachhead specialty confirmation.
- Pricing validation: does pilot data support R$99–149 or push higher/lower?
- How much of P-031–P-064 plugin architecture is demo-visible vs. post-hackathon?
