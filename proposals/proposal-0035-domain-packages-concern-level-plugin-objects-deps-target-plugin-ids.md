+++
id = "P-035"
title = "Domain packages, concern-level plugin objects: deps target plugin ids"
status = "open"
created = "2026-11-09"
owner = "devin"
priority = "medium"
epic = "platform"
+++

## Context

Supersedes P-034 (package-per-concern rejected). The plugin contract
already models each contribution as its own plugin *object* with its
own id (`patients.tables`, `patients.services`, `patients.routes`,
`patients.landing-nav`, `patients.panel`) inside a single domain
package (`plugins/patients`). The missing piece is making cross-plugin
integration explicit: deps should target concern-level ids, and
`optionalDeps` is the mechanism for soft integration.

## Proposal

Keep package-per-domain. Formalize two conventions:

1. **Id convention**: `<domain>.<concern>` — already in use
   (`patients.services`, `platform.admin`). The concern suffix
   matches the contribution kind.

2. **Deps target plugin ids, not packages.** A plugin in ANY package
   can `requiredDeps`/`optionalDeps` on any concern object:

   ```ts
   // plugins/platform-sidebar — shows platform nav only if the
   // console shell plugin is present
   definePlugin({
     id: "platform-sidebar",
     optionalDeps: ["platform.routes"],
   })

   // plugins/dashboard — patient stats only if patients is installed
   definePlugin({
     id: "dashboard.stats",
     optionalDeps: ["patients.services"],
   })
   ```

   `optionalDeps` = "integrate if present, skip silently if not" —
   exactly the decoupling needed for optional feature plugins. This
   is already implemented in `orderPlugins` (soft edges used when the
target exists); this proposal ratifies it as THE integration
   mechanism and documents the pattern.

## What this gives us

- A domain package stays cohesive (schema + services + views +
  surfaces together) while every concern remains an independent,
  addressable plugin.
- Feature combos compose without edits: install `plugins/patients` →
  tables+services+routes+nav+panel all register; another plugin can
  optionally hang off `patients.services` or `patients.panel`.
- No package explosion; discovery glob unchanged.

## Work items

- Document the `<domain>.<concern>` id convention + optionalDeps
  integration pattern (AGENTS.md or kernel README).
- Audit existing plugins: add `optionalDeps` where integration is
  actually soft (e.g. dashboard → domain services for stat cards).
- Optional: `PluginRegistry` typing already covers dep ids — verify
  `optionalDeps` entries are registry-checked too.

## Non-goals

- No package-per-concern split (P-034 rejected).
- No wildcard/group deps (`patients.*`) — explicit ids only.
- No runtime plugin loading — still static composition.
