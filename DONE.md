# Done

- T-069 product: fix failing test PatientFormPage 'an exact date replaces the age' (Value "May" not found in options; plugins/patients/src/views/PatientFormPage.test.tsx) — 1/703 red, breaks CI · pri:high · owner:human:emanuel · created:2026-11-09 · #github-actions · blocks:T-030
- T-070 storybook: build fails — @ktbhealth/ui/components/job-status unresolved in src/blocks/JobsBoard.tsx (pinned ui#v0.1.2 lacks it); tag new ui release + bump dep · pri:high · owner:human:emanuel · created:2026-11-09 · #github-actions · blocks:T-030
- T-071 product: commit package.json trustedDependencies (onnxruntime-node, protobufjs) — formatted with vp fmt, needed for vp check to pass · owner:human:emanuel · created:2026-11-09 · #github-actions · blocks:T-030
- T-072 ui: remove duplicate "vue" dependency in package.json (lines 69 and 86, bun install warns) · pri:low · owner:human:emanuel · created:2026-11-09 · #github-actions · blocks:T-030
- T-038 Platform panel host: plugins/platform owns /platform shell + sidebar slot (P-031) · pri:high · owner:devin · created:2026-11-09 · #admin
- T-039 Kernel: Plugin.panel.sidebar contribution field for panel nav (P-031) · pri:high · owner:devin · created:2026-11-09 · #admin
- T-040 Patients plugin: add /platform/patients item to platform panel sidebar (P-031) · owner:devin · created:2026-11-09 · #admin
- T-001 Cover slide: add real team photos and names — 4 people. Currently placeholder circles; swap in photos and real names before recording. · owner:human:emanuel · created:2026-11-08
- T-068 product: main is red on the coverage gate (99.77% lines) after the plugin-owned routes / server-fns refactor — 8 files below 100%: routes/api/bots.ts, server-fns/runtime.ts, consultations fns.ts + index.ts + api-routes/patients.$id.export.ts, patients fns.ts + guard.ts + routes/new.tsx · pri:high · owner:human:emanuel · created:2026-11-09 · #coverage · #github-actions · blocks:T-059
