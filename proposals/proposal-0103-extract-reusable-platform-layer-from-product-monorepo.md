+++
id = "P-103"
title = "Extract reusable platform layer from product monorepo"
status = "open"
created = "2026-11-19"
owner = "devin"
priority = "high"
labels = ["architecture", "platform"]
links = ["T-059", "T-064", "P-084", "P-094", "P-101"]
+++

## Summary

Decouple `ktbhealth/product` into a reusable platform layer and a product layer where KTB Health is the first embedded application — a seed plugin-pack plus custom clinical plugins.

## Current state

The monorepo is already close to this shape:

- DB schema is per-plugin: `packages/db/src/schema.gen.ts` is codegen aggregating `plugins/<pkg>/src/schema.ts`; host-owned slices (`auth`, `plugin-state`, `job-runs`) are all generic.
- Plugin deps are id-based (`patients.services → consultations`), not package-based.
- `plugins/portal` verified domain-free: deps are only `@ktb/auth`, `authz`, `i18n`, `kernel`, `platform` + `@ktbhealth/ui`. It is a generic audience-gated portal shell — belongs to platform. `patient-portal`/`patient-access` remain product plugins that instantiate it.

## Proposed split

### `@ktbhealth/platform` repo (reusable kernel)

- Kernel: `kernel`, `dsl`, `driver-framework`
- Generic ports + adapters: `db`, `db-sqlite`, `auth`, `authz`, `tenant`, `email*` (port + cloudflare/smtp/stub), `jobs-port`, `jobs-transport-redis`, `captcha*`, `crypto*`, `observability*`, `vault-keyring`, `i18n`, `server-fns`, `json-schema`, `seed`, `eval-core`, `eval-adapter-local/stub`
- Host infra: `platform`, `app-host` (after decoupling — see blockers), `packages/rbac` + `plugins/rbac` (audit first)
- Generic plugins: `pages`, `forms`, `rich-text`, `views`, `wizards`, `flows`, `triggers`, `kv`, `storage-dataset`, `notifications`, `audit`, `invites`, `users`, `groups`, `theme`, `site`, `dashboard`, `jobs`, `vault`, `portal`, `audience` (audit), `platform-header`, `manager` (audit), `api` (audit)
- Arch-lint oxlint plugin (`@ktbhealth/workspace` eslint-config-arch) — every consumer needs it
- Distribution: git-tag deps (`github:ktbhealth/platform#vX.Y.Z`), same pattern as `ui`

### `ktbhealth/product` repo (embedded app)

- Clinical plugins: `patients`, `consultations`, `doctors`, `clinician`, `secretaries`, `agenda`, `patient-portal`, `patient-access`, `credentials`, `document-store`, `transcription-asr/tts`, `ai-model/ai-prompt/ai-provider`, `llm-eval`, `eval-persona`, `eval-validator`, `interop`, `interop-fhir`, `interop-openehr`, `machines` (audit)
- Healthtech ports/adapters: `transcription*`, `anamnesis*`, `interop-*`, `ai-adapter-*`, `eval-adapter-cloud` (audit), `bot*` (audit)
- `apps/web`, `apps/jobs-worker`, `apps/cli` — composition roots wiring platform + clinical pack
- `infra/` (cloudflare, oci, lgtm, ports, ci-hook), `evals/`, `tools/` domain tooling

## Blockers

1. **`app-host` is a composition root in package form** — imports ~35 packages including domain ones (`anamnesis`, `api`, `document-store`, `flows`, `views`, `wizards`). Must become manifest-driven: accept plugin manifests/registration input; each app injects its own pack. This refactor gates everything.
2. **Straddling plugins need audit**: `portal` (verified platform), `audience`, `rbac`, `manager`, `api`, `machines`, `bot*` — classify via dep graph.
3. **Shared generated artifacts**: `routeTree.gen.ts`, paraglide i18n output, drizzle migrations — decide per-repo generation vs platform-owned codegen.

## Sequence

1. Audit plugin dep graph → concrete platform/product lists (this proposal's table is the draft).
2. Refactor `app-host` → manifest-driven registrar; move KTB registrations into `apps/web`.
3. Extract generic packages to `platform` repo; product consumes via git-tag deps.
4. `product` repo keeps clinical plugins + apps + infra (mostly deletion of generic packages).
5. `project`/myco remains the cross-repo ledger; ADRs for the boundary decisions land here.

## Open questions

- Does `rbac` belong to platform (generic grants) or product? P-084/P-101 (permission system, ReBAC) will shape this.
- Single `platform` repo vs split `kernel`/`platform`/`ui`-style repos per concern?
- Versioning cadence: tag per extraction milestone vs semver per package?
- Do evals (`evals/`, `llm-eval`) stay product-only or become a platform capability other apps reuse?

## Relations

- P-094 (dynamic runtime platform) — this extraction is the physical precondition for schema-driven runtime plugins.
- P-084 / P-101 — authz/ReBAC decisions affect where `rbac`/`authz` land.
- T-059/T-064 — cross-repo CI must handle platform↔product git-tag deps (same problem as `ui`).
