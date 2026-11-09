+++
id = "P-031"
title = "Platform panel host: admin console as plugin with sidebar extension API"
status = "open"
created = "2026-11-09"
owner = "devin"
priority = "high"
labels = ["admin"]
epic = "admin"
+++

## Idea

Make the admin panel (P-025/P-026) a first-class plugin — `plugins/platform`
— that owns the `/admin` shell *and* a sidebar contribution slot. Other
plugins declare admin-panel nav entries and admin routes through the
plugin contract instead of the host hardcoding admin pages.

Today `Plugin.nav` feeds the top-level doctor-facing header only
(`__root.tsx` flatMaps `p.nav`). There is no mechanism for a plugin to
add an item to the admin sidebar, and admin views would otherwise have
to be wired by hand in `apps/web` for every plugin — exactly the
coupling the plugin contract exists to avoid.

## Shape

- **`@ktb/kernel`**: extend `Plugin` with an optional panel contribution,
  e.g. `panel?: { sidebar?: ReadonlyArray<{ to: string; label: string; icon?: … }> }`
  plus the admin route subtree already covered by `routes`.
- **`plugins/platform`**: owns `/admin` route group (shell, dense
  sidebar layout, server-side admin role gate from P-026), renders the
  aggregated `panel.sidebar` items grouped by plugin, and hosts the
  shared admin primitives (tables, status badges, audit hooks).
- **Consumers**: `plugins/patients` adds e.g. `/admin/patients` to the
  panel sidebar; consultations and future plugins (ops health, billing,
  audit views per P-027–P-030) do the same.
- **Ordering**: sidebar entries sort by plugin topo order (`deps`) so
  e.g. patients' admin views sit next to related entities.

## Why it matters

P-025–P-030 scope admin views per capability, but without a panel-host
contract each of those becomes bespoke host wiring. Defining the
sidebar slot now means T-032..T-037 implement *views*, not shell
plumbing — and any future plugin gets admin surfacing for free by
declaring one array.

## Open questions

- Nav item shape: does the admin sidebar need icons/badges, or is
  `{ to, label }` enough to start (match existing `nav`)?
- Does `panel.sidebar` live on the same `Plugin` interface, or a
  separate `admin` contribution object so non-admin plugins stay
  untouched? (Leaning: separate optional field, same interface.)
- i18n: plugin nav labels currently resolve via `pluginNavLabel` —
  admin labels need the same treatment (blocks T-025-style debt).

## Non-goals

- No dynamic/runtime plugin loading — static registry stays (workerd).
- No change to doctor-facing `nav` rendering.
