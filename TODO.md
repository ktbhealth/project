# TODO — ktbhealth org board

- [ ] T-001 Cover slide: add real team photos and names — 4 people. Currently placeholder circles; swap in photos and real names before recording. · created:2026-11-08
- [ ] T-017 HackBrasil: record and upload presentation video (~3 min) · pri:high · owner:human:team · created:2026-11-07 · epic:hackbrasil
- [ ] T-025 Localize consultations plugin views (NewConsultationPage, ConsultationDetailPage, ConsultationsPage ~1.5k strings) — needs paraglide output or messages port reachable from plugins/ (arch lint blocks plugins → apps imports) · pri:low · created:2026-11-08
- [ ] T-026 Set up peregrin as CI · pri:high · created:2026-11-08
- [ ] T-028 Update branding: new logo, favicon, and related assets · owner:human:charles · created:2026-11-08
- [ ] T-030 Configure GitHub Actions CI; later integrate Peregrin as CI orchestrator inside GHA · owner:human:emanuel · created:2026-11-08 · #github-actions
- [ ] T-031 Platform QoL features around core consult flow: scheduling/agenda, reminders, patient follow-up nudges, quick notes — doctor-facing conveniences that increase daily value beyond documentation · owner:human:vini · created:2026-11-08 · #platform-qol
- [ ] T-032 Admin panel: system console implementation (P-025 umbrella) · pri:high · created:2026-11-08 · #admin · epic:admin
  - [ ] T-033 Admin shell & authz: /admin route group, server-side admin role check (env allowlist first), dense sidebar layout (P-026) · pri:high · created:2026-11-08 · #admin · blocks:T-032 · epic:admin
  - [ ] T-034 Admin entity views: users/orgs/patients tables + detail, suspend/reactivate actions (P-027) · created:2026-11-08 · #admin · blocks:T-032 · epic:admin
  - [ ] T-035 Admin ops health: queue/job status, webhook delivery, storage usage, failed-job retry (P-028) · created:2026-11-08 · #admin · blocks:T-032 · epic:admin
  - [ ] T-036 Admin billing views: subscriptions, invoices, plan overrides (P-029, needs P-024) · created:2026-11-08 · #admin · blocks:T-032 · epic:admin
  - [ ] T-037 Admin audit log: admin\_audit table + write helper + /admin/audit view (P-030) · created:2026-11-08 · #admin · blocks:T-032 · epic:admin
- [ ] T-038 Platform panel host: plugins/platform owns /admin shell + sidebar slot (P-031) · pri:high · owner:devin · created:2026-11-09 · #admin
- [ ] T-039 Kernel: Plugin.panel.sidebar contribution field for admin nav (P-031) · pri:high · owner:devin · created:2026-11-09 · #admin
