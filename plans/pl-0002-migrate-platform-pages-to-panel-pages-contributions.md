+++
id = "PL-002"
title = "Migrate /platform pages to panel.pages contributions"
status = "done"
created = "2026-11-09"
priority = "med"
+++

Implements P-050. Ordered steps:

1. **kernel** (`packages/kernel/src/index.ts`): add `PanelPage` type — `{id, path, component: ComponentType<{data: unknown}>, loader?: () => Effect<unknown, unknown, unknown>}` — and `pages?: readonly PanelPage[]` on PanelPlugin. Generic-typed per plugin at use site.

2. **host registry + routes** (`apps/web`):
   - `src/lib/platform-pages.ts`: collect `panel.pages` from discovered plugins into `Map<path, PanelPage>`.
   - `src/routes/platform/index.tsx`: rewrite as dispatcher — look up `/platform`, run loader via `runAs`, render component; `notFound()` when absent.
   - `src/routes/platform/\$page.tsx` (new): same for `/platform/<param>`.
   - Keep the existing layout/admin gate untouched.

3. **migrate pages** (each domain plugin's `*.panel` plugin object gains `panel.pages`):
   - patients → `/platform/patients`, loader `Effect.gen(yield* Patients → adminList())`
   - consultations → `/platform/consultations`, `Consultations.adminList()`
   - users → `/platform/users`, `Users.adminList()`
   - groups → `/platform/groups`, `Groups.list()`
   - dashboard → `/platform` index, aggregate Effect (Patients.adminList + Consultations.adminList + serviceOption(Notifications) for activity card)
   - Delete `platform/{patients,consultations,users,groups}.tsx`; index.tsx becomes the generic dispatcher.

4. **cleanup**: remove now-dead `adminList*Fn` platform-page fns from `lib/fns.ts` if unused elsewhere; keep notification/jobs fns (still used).

5. **tests**: platform route tests updated for dispatcher shape; per-plugin index.test asserts page descriptors; apps/web suite green; `vp check` 0 errors; dev server verify.

Watch: loader Effect error types must stay `unknown`-compatible for the host boundary; component `data` prop typing per plugin wrapper.
