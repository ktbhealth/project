+++
id = "P-050"
title = "Platform pages as plugin contributions"
status = "done"
created = "2026-11-09"
priority = "medium"
+++

## Context

The /platform console is the product surface for now — every domain's admin UI lives there. Today the split is half-and-half: domain plugins own services, CQRS APIs, views, and sidebar entries, but each console page is a physical route file in apps/web/src/routes/platform/ plus a dedicated server fn in lib/fns.ts. The plugin never owns its page contract end-to-end.

## Proposal

Add `panel.pages` to the panel contribution: a plugin declares console pages as data, not files.

```ts
panel: {
  sidebar: [{ to: "/platform/patients", ... }],           // unchanged
  pages: [{
    id: "patients.admin",
    path: "/platform/patients",
    component: AdminPatientsPage,      // receives { data }
    loader: () => Effect.gen(...),     // Effect — host runs it via runAs
  }],
}
```

- `loader` is an `Effect` value, not a server fn — the plugin needs zero host imports. The host route runs it inside `runAs(sessionUser, ...)`, so service-level authz (PlatformAdmin gate, requirePrincipalUser) applies unchanged.
- The host keeps exactly TWO generic physical routes: `platform/index.tsx` (path `/platform`) and `platform/$page.tsx` (everything else). Each looks up the page descriptor in the plugin registry, runs the loader, renders the component. No per-domain files, no per-page fns.
- Dashboard's aggregate loader (patients + consultations + notifications) becomes a single Effect in the dashboard plugin using `Effect.serviceOption` for the optional notifications backend — cleaner than today's four-fn Promise.all.
- Mutations stay on the existing injected-api pattern; admin pages are read-only today anyway.
- Sidebar stays a separate `panel.sidebar` contribution (groups/labels/icons are nav concerns, pages are route concerns).

## Why this shape

- Same pattern as `panel.headerActions` (component contributions) and `panel.sidebar` — the panel vocabulary just gains its third member.
- Port-adapter: plugin = page + data logic; host = routing, session, auth boundary, render shell.
- Optional plugins degrade gracefully: absent plugin → no page descriptor → 404 via notFound().
- Sets up P-033 (self-contained plugin routes) later without committing to virtual-file-route limitations now.

## Open decisions

- Nested paths (`/platform/foo/bar`): spec supports string paths; start with a `$page` splat or exact-match map — decide at implementation.
- Page-level `beforeLoad`-style guards: deferred — service authz inside loader Effects covers current needs.
