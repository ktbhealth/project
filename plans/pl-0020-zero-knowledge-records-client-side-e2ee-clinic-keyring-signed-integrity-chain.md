+++
id = "PL-020"
title = "Zero-knowledge records: client-side E2EE, clinic keyring, signed integrity chain"
status = "done"
created = "2026-11-15"
owner = "devin"
priority = "high"
labels = ["security", "zk"]
epic = "zero-knowledge"
+++

## Zero-knowledge records — client E2EE + integrity chain

**Model A (full ZK).** All record content is encrypted in the browser;
the server stores ciphertext plus the metadata authz needs
(ids, ownerId/tenantId, status, timestamps, consent). Keys live with
the clinics: each tenant (`groups.id`) holds an ECDH P-256 wrap pair +
ECDSA P-256 signing pair; a solo doctor (`tenant_id` NULL) is a clinic
of one. Web Crypto API throughout (`globalThis.crypto.subtle` is
isomorphic across browser / workerd / Node ≥20 and passes the arch
policy — no `node:*`, no `process`).

### F0 — `@ktb/crypto` package (primitives + subpaths)

- [ ] `aead`: AES-256-GCM encrypt/decrypt, versioned envelope
      `v1.<b64url(iv)>.<b64url(ct)>`
- [ ] `wrap`: ECDH P-256 + HKDF-SHA-256 → DEK wrap/unwrap (ECIES-style)
- [ ] `sign`: ECDSA P-256 sign/verify of document hashes
- [ ] `digest` + `canonicalize`: SHA-256 over stable JSON (sorted keys,
      NFC) — the legal integrity hash input
- [ ] `kdf`: PBKDF2-SHA-256 ≥600k iters (generic primitive only —
      recovery encodes the key, it is not passphrase-derived)
- [ ] chunked AES-GCM helpers for large blobs (audio)
- [ ] `/cipher` subpath: declarative entity specs —
      `defineCipher({ entity, fields, plaintext })` →
      `encryptRow` / `decryptRow` / `encryptPatch`
- [ ] `/keyring` subpath: client vault — non-extractable device
      CryptoKeys persisted in IndexedDB, clinic master unwrap, session
      DEK cache
- Verify: `cd packages/crypto && bunx vitest run` (100% coverage gate)

### F1 — Integrity chain (works before ZK, upgraded by it)

- [ ] `document_versions` table (append-only): consultation_id,
      version, content_hash, signature (NULL until clinic key),
      prev_hash, ciphertext, actor_user_id, engine, created_at
- [ ] `consultations`: + anamnesis_hash, anamnesis_version
- [ ] Interim: server computes content_hash on save/attach —
      tamper detection works today; ZK tenants later sign the same
      hash client-side (ECDSA, `signature` fills in)
- [ ] `blob_hash` over the stored ciphertext for corruption detection
      (distinct from content_hash over canonical plaintext)
- [ ] `admin_audit.prev_hash` — tamper-evident admin log chain
- [ ] `consultations.verifyIntegrity` server fn + integrity badge on
      ConsultationDetailPage; verify re-runs client-side for ZK rows
- [ ] Backfill: existing anamneses → version 1 hashes

### F2 — Clinic keyring (`plugins/vault`)

- [ ] `clinic_keys`: clinic pubkeys + master key wrapped to each
      approved member-device pubkey
- [ ] `entity_deks`: per-record AES-256-GCM DEK wrapped per authorized
      reader (clinic members; patient via portal grant)
- [ ] Onboarding: keypair generation, recovery phrase ENCODING the
      master key (printable, clinic-held; lost key = lost data),
      admin approval of member devices
- [ ] `groups`: + zk flag — per-tenant rollout, non-ZK tenants keep
      today's flow during migration

### F3 — Entity migration + client pipeline

- [ ] `patients`: all PII fields ciphered (name, socialName, phone,
      email, birthDate, address*, motherName, gender, maritalStatus,
      raceColor…); ids/flags stay plaintext
- [ ] `consultations`: transcript/anamnesis/review/serverTranscript/
      correctedTranscript ciphered; `fns.ts` wire schema accepts
      opaque ciphertext (was AnamnesisSchema); hasAnamnesis derives
      from column presence; versions signed client-side
- [ ] audio: optional client-encrypted R2 upload (chunked AES-GCM);
      ZK tenants skip the server Whisper pipeline entirely
- [ ] client consult pipeline: webspeech transcript → clinic BYOK LLM
      call → client-side patientHistory/factContent → sign + encrypt
      + store; PDF render moves client-side
- [ ] patient-portal: clinic wraps consultation DEK to the patient's
      key during the invite flow; portal decrypts in the browser
- [ ] notifications: metadata-only bodies under ZK (server cannot
      read names — no PHI in title/body/payload)
- [ ] stays plaintext BY DESIGN (operational metadata): auth emails,
      delivery addresses (invites.email / patient_access.email /
      secretaries.email — SMTP needs them), groups.name/slug, every
      id/status/timestamp used by authz

### F4 — Availability

- [ ] D1 PITR/Time Travel + tested restore runbook
- [ ] ciphertext backup/export story per tenant
- [ ] DEK re-wrap ceremony on member revocation (client-side)

### Corrections from plan review (v1 → v2)

- Ordering: Fase 1 no longer assumes the clinic key — hash chain is
  server-computed interim, `signature` fills in when F2 lands
- Blind indexes REMOVED: search/dedup is client-side under ZK, and
  invite/access emails must stay plaintext for SMTP anyway
- Package shape: one `@ktb/crypto` with `/cipher` + `/keyring`
  subpaths instead of three packages (repo subpath convention)
- `content_hash` (plaintext, signed) vs `blob_hash` (ciphertext,
  server) now distinguished
- `fns.ts` wire schema must accept opaque ciphertext for ZK rows
- Notification bodies go metadata-only — they leak names today
- Audio upload is optional evidence for ZK tenants (webspeech never
  needs a server step); it is never server-transcribable

### Open decisions

- LLM extraction under ZK: clinic BYOK (purest — browser calls the
  provider with the clinic's own key from its vault) vs KTB stateless
  relay (plaintext in-flight only, documented residual trust)
- Rollout: per-tenant `zk` flag coexistence vs hard migration
