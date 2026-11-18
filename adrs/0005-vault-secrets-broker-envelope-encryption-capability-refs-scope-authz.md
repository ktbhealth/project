+++
id = "ADR-0005"
title = "Vault secrets broker: envelope encryption, capability refs, scope authz"
status = "open"
created = "2026-11-18"
owner = "devin"
priority = "med"
links = ["P-098"]
+++

## Context

P-098 introduced `plugins/vault` as the canonical credentials/secrets broker: `vault_secret` rows hold a per-row DEK (AES-256-GCM envelope) wrapped under the deployment KEK (`VAULT_MASTER_KEY`, env driver), with AAD binding both envelopes to `(scope, name)`. Consumers reference secrets as `vault:<id>` — never raw material. `Secrets.resolve` is the only plaintext read-back and is deliberately ungated (the ref is the capability); every lifecycle op is audit-logged.

C4/C5 hardened the surface: canonical `vault:<digits>` ref spelling, resolve audit coverage test, machines `secretRef` format validation, and beat rate limiting.

## Decision

1. **Envelope encryption, not field-level**: per-row DEKs wrapped under one deployment KEK. Rotating a secret re-mints the DEK; rotating the KEK is a future keyring-version operation (`kekVersion` column already records the fingerprint).
2. **`resolve` is a capability, not an RPC**: callers (SMTP driver, AI endpoint descriptor resolution, credentials facade) run server-side in system context. User-facing surfaces must gate themselves; the wire never exposes `resolve`.
3. **Scope model**: `"platform"` (PlatformAdmin), `"<tenantId>"` (bound-tenant member read / admin write), `"user:<id>"` (self only — powers the credentials facade).
4. **Audit**: create/update/resolve/rotate/remove all recorded; `resolve` carries actor attribution via the bound Principal. Detail payloads are ids/metadata only — enforced by convention + test.
5. **Fail closed**: absent `VAULT_MASTER_KEY` → `cipher-unavailable`; tampered envelopes → typed `decrypt-failed`, never silent null; disabled rows never resolve.
6. **ZK orthogonality**: vault secrets are infrastructure credentials (API keys, tokens), not PHI. Zero-knowledge record encryption is client-side and unaffected; no secret value may be embedded into record plaintext by any consumer.

## Consequences

- `credential:<id>` (legacy apiKey store) remains as a read/resolve path during migration; new writes go to vault.
- SMTP/CAPTCHA stay env-selected drivers — they resolve before plugin services bind; vault migration there is a documented non-goal for now.
- Rate limiting on `resolve` was considered and rejected: it is an internal port called by trusted adapters at request time; abuse would surface as decrypt CPU, which is negligible vs. the authz-gated write surface.
- `machines.secretRef` accepts `vault:<id>` (validated) and other schemes pass through; consumers resolve.
