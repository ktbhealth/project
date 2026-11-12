+++
id = "P-031"
title = "Platform panel host: console as plugin with /platform shell and sidebar API"
status = "in-progress"
created = "2026-11-09"
owner = "devin"
priority = "high"
labels = ["admin"]
epic = "admin"
+++

## Idea

Make the admin panel (P-025/P-026) a first-class plugin — `plugins/platform`
— that owns the `/platform` shell *and* a sidebar contribution slot. Other
plugins declare panel nav entries and console routes through the
plugin contract instead of the host hardcoding console pages.

Today `Plugin.nav` feeds the top-level doctor-facing header only
(`__root.tsx` flatMaps `p.nav`). There is no mechanism for a plugin to
add an item to the panel sidebar, and console views would otherwise have
to be wired by hand in `apps/web` for every plugin — exactly the
coupling the plugin contract exists to avoid.

## Shape

- **`@ktb/kernel`**: extend `Plugin` with an optional panel contribution,
  e.g. `panel?: { sidebar?: ReadonlyArray<{ to: string; label: string; icon?: … }> }`
  plus the console route subtree already covered by `routes`.
- **`plugins/platform`**: owns the `/platform` route group (shell, dense
  sidebar layout, server-side admin role gate from P-026), renders the
  aggregated `panel.sidebar` items grouped by plugin, and hosts the
  shared console primitives (tables, status badges, audit hooks).
- **Consumers**: `plugins/patients` adds e.g. `/platform/patients` to the
  panel sidebar; consultations and future plugins (ops health, billing,
  audit views per P-027–P-030) do the same.
- **Ordering**: sidebar entries sort by plugin topo order (`deps`) so
  e.g. patients' console views sit next to related entities.

## Why it matters

P-025–P-030 scope console views per capability, but without a panel-host
contract each of those becomes bespoke host wiring. Defining the
sidebar slot now means T-032..T-037 implement *views*, not shell
plumbing — and any future plugin gets console surfacing for free by
declaring one array.

## Open questions

- Nav item shape: does the panel sidebar need icons/badges, or is
  `{ to, label }` enough to start (match existing `nav`)?
- Does `panel.sidebar` live on the same `Plugin` interface, or a
  separate `panel` contribution object so non-admin plugins stay
  untouched? (Leaning: separate optional field, same interface.)
- i18n: plugin nav labels currently resolve via `pluginNavLabel` —
  console labels need the same treatment (blocks T-025-style debt).

## Non-goals

- No dynamic/runtime plugin loading — static registry stays (workerd).
- No change to doctor-facing `nav` rendering.
