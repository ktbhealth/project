+++
id = "P-081"
title = "Portal plugin: audience-scoped portal instances with dynamic sidebars (doctor/secretary/translator/platform)"
status = "done"
created = "2026-11-15"
priority = "med"
links = ["P-031", "P-050", "P-058", "P-059", "P-060", "P-070"]
done = "2026-11-17"
+++

## Context

Today there are three disconnected surface patterns:

- **App shell** (`__root.tsx`): flat top-nav driven by `app.header.*` chrome regions. The doctor/secretary experience lives here — consultations, patients, doctor select, secretaries — as a row of links that doesn't scale and can't express groups, per-role visibility, or dense workflows.
- **`/platform` console** (`platform.tsx` + `PlatformConsoleShell`): the one true sidebar shell. Plugins fill `console.sidebar` / `console.header.actions` chrome regions and declare `panel.pages` under `/platform/*`. Gated hard by `platformAdminFn` (env allowlist). This is exactly the "dynamic sidebar" pattern, but it has exactly ONE instance and ONE audience (platform admin).
- **`/portal`**: the patient portal — a hardcoded, non-plugin-driven subtree.

The ask: generalize the console pattern into **portal instances** — named, audience-scoped shells where each instance renders only the sidebar groups/items and pages meant for its audience. Doctor portal, secretary portal, translator portal, platform-manager portal are all instances of the same machinery. `/platform` becomes just the admin instance; the patient portal is the read-only instance; the doctor/secretary app shell migrates onto it.

## Proposal

### 1. Portal instance registry

A portal instance is data:

```ts
interface PortalInstance {
  id: string;                    // "clinician", "secretary", "translator", "platform", "patient"
  basePath: string;              // "/app", "/portal/doctor", "/platform", …
  label: string;                 // shell title / breadcrumb root
  audience: PortalAudienceRule;  // who may enter — see §3
  icon?: ComponentType;
}
```

Instances are contributed like everything else — a `portal.instance` contribution (generic `contributions` entry or a dedicated field per P-070) so feature plugins can ship their own portal (translator, billing, …) without touching the host. `plugins/portal` (new) owns the registry, the shell view, and the audience port; `packages/` stays infra-only per the packages/plugins rule — the shell view likely belongs in `@ktb/ui` or a thin `plugins/portal/src/views/` like `platform-sidebar` does today.

### 2. Instance-scoped chrome + pages

Extend the existing mechanisms rather than inventing parallel ones:

- **Chrome**: sidebar/header contributions gain an optional `portal` (or `instance`) field — or equivalently new regions `portal.sidebar` / `portal.header.actions` filtered by instance at collection time. `when.paths` keeps working for sub-path filtering; the instance tag is what replaces today's implicit "console = admin" coupling.
- **Pages**: `Page`/`PanelPage` gain an optional portal scoping (`context: "portal:<id>"` or a `portalId` field). The existing splat dispatchers (`platform/$`, `$.tsx`) generalize to resolve `basePath + path` per instance. Wildcard `/*` pages keep working per instance.
- **Shell**: `PlatformConsoleShell` generalizes into `PortalShell` — same SidebarProvider layout, breadcrumb header, sign-out footer — parameterized by instance label/icon and its item set. `/platform` keeps working by registering `platform` as an instance with `audience: platform-admin`.

### 3. Audience resolution (the new port)

`when` rules are pathname-only today; portals need principal-aware filtering. Introduce a `PortalAudience` port (`@ktb/portal` or `packages/authz` extension):

```ts
class PortalAudience extends Context.Service<...> {
  // memberships of the current principal, e.g.
  // { "clinician": true, "secretary": [doctorIds…], "platform-admin": false }
  memberships(): Effect<ReadonlySet<string> | AudienceMap>
}
```

Backed by existing ports: `DoctorProfiles`/`DoctorDelegations` (clinician vs secretary), `PlatformAdmin` (platform instance), patient links (`patient-access`), and RBAC subject roles for future audiences (translator, billing). Two enforcement points, same as today:

- **Route guard**: the portal layout route runs the instance's `audience` check server-side → `notFound()` for non-members (never just hidden nav).
- **Contribution filter**: the shell collects only items/pages tagged for instances the user belongs to; untagged items default to the "app" audience or are excluded — decide in review.

### 4. Migration path

1. Land `plugins/portal` + `PortalShell` + audience port; register `platform` as an instance pointing at the existing `/platform` tree — zero behavior change, proves the abstraction.
2. Register `patient` instance wrapping the existing `/portal` routes (or leave `/portal` as-is initially — it's already effectively an instance).
3. Register `clinician` instance; move doctor-facing `app.header.nav` links to `portal.sidebar` items; decide whether the clinician portal keeps root-relative paths (`/consultations`, `/patients`) or moves under `/app/*`. Root-relative is less disruptive; a `basePath: "/"` instance with sidebar chrome replacing the top nav is the smallest diff.
4. Secretary portal: same shell, `audience` = active `doctor_access` delegation; items filtered to secretary-safe surfaces (schedule, patients, consult drafts — not secretaries-management, which stays clinician-only per the existing `Forbidden`-under-delegation rules).
5. Translator / future portals: new instance + tagged contributions, no host changes.

## Why this shape

- Reuses proven machinery: chrome regions (P-060), generic pages (P-059), panel dispatch (P-050), contribution kinds (P-070), plugin enable/disable (P-058 — portals can be disable-able per tenant later).
- Audience scoping is data (`audience` rule + contribution tags), not code — new portals are plugin contributions, matching the registry-citizen rule.
- Authz stays inside services; the portal guard is a UX gate (404), never the security boundary — same contract as `panel.pages[].guard`.
- Positions P-065: specialty-first scribe needs locale/specialty-pack-aware surfaces; portal instances are where `specialty.*`/`locale.*` packs (P-091) can scope sidebar groups per clinic configuration later.

## Open decisions

- **Path strategy for the clinician portal**: root-relative (`/` instance swapping top-nav for sidebar) vs namespaced (`/app/*`, `/portal/doctor/*`). Root-relative is the smaller rewrite; namespaced is cleaner for multi-portal users (a user who is both clinician and translator needs a switcher — header portal switcher becomes necessary).
- **Static vs DB-driven instances**: start static (contributed descriptors); DB-backed instances (tenant-configurable portals) can ride `pages`-style wildcard resolution later.
- **`console.*` regions**: keep as the platform instance's alias, or migrate all regions to `portal.*` and treat `/platform` as one instance among several.
- **Patient portal**: fold into the instance model now or defer — it's the odd one out (no sidebar today, patientOnly gating in `__root`).
- **Portal switcher UX**: when a principal holds multiple memberships, does the shell show a switcher (like doctor select does today via `DoctorProfiles`)?
