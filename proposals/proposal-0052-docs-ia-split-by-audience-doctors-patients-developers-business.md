+++
id = "P-052"
title = "Docs IA: split by audience (doctors/patients/developers/business)"
status = "open"
created = "2026-11-10"
priority = "medium"
labels = ["docs"]
+++

## Proposal

Restructure ktbhealth/docs content from Diátaxis-top-level to audience-top-level:

- `doctors/` — clinical users: getting-started, consult-flow, transcription-faq, troubleshooting
- `patients/` — care recipients: what-to-expect, privacy-and-data (STUB or fold into doctors/ if no patient-facing surfaces exist yet — don't document vaporware)
- `developers/` — builders: keeps Diátaxis inside (tutorials/how-to/reference/explanation) — existing content lands here mostly as-is
- `business/` — partners/enterprise/investors: positioning, security-and-compliance, deployment-options, contact
- `index.mdx` becomes an audience picker; fumadocs tabs render the audience switcher in chrome

Rationale: audience answers "who is asking"; Diátaxis 4x4 matrix too sparse. developers/ is where Diátaxis maps naturally; business/ is cheap high-value new content (positioning/security/deploy story exists implicitly in codebase + ADRs).

Open question: patients/ only if real patient-facing surfaces exist.
