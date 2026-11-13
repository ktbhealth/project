+++
id = "P-006"
title = "Patient history sharing across clinics (consent-scoped)"
status = "open"
created = "2026-11-08"
priority = "med"
labels = ["product", "v2"]
+++

## Idea

Patient history sharing: the patient authorizes (same LGPD
consent as capture) that other clinics/physicians access their
consults — KTB's "portable medical record".

## Surface

- Per-patient consent scope (read vs. write)
- Access link/token with expiration
- Audit of who accessed what
- Multi-tenant: data from different clinics on the same patient

## Cautions

- LGPD: granular, revocable consent; minimization
- Never share data between physicians without explicit consent
- Start read-only; shared write is a later phase

## Depends on

Mature patient↔consults model.
