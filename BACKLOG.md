# Backlog

- T-021 Copy/legal positioning: remove any language that induces or suggests the doctor should decide; KTB is a platform/tool that helps doctors document, never decides. Update draft-landing copy, web home copy, and any feature copy/code that implies decision support. Redeploy draft-landing and web after. · created:2026-11-08
- T-013 Desktop app (Tauri): SystemAudioTranscriber via loopback to capture ANY call app (WhatsApp Desktop, Zoom, Teams), pro tier, post-pilot · pri:low · owner:devin · created:2026-11-06 · #audio · epic:mvp
- T-029 Buy a custom domain for KTB Health services (zone in the KTB Health Cloudflare account), then attach it to the web/storybook/landing Workers · owner:human:luis · created:2026-11-08 · #infra
- T-027 Onboard luisfuturist.com zone to Cloudflare account, then re-enable domain in storybook/alchemy.run.ts (storybook.ktb.luisfuturist.com) · created:2026-11-08
- T-067 CI: scheduled nightly run on main (e2e + audit) to catch breakage from dependencies · pri:low · owner:human:emanuel · created:2026-11-09 · #ci-hardening · #github-actions
- T-043 Upgrade @effect-server-utils/cqrs to stable · created:2026-11-09
- T-044 Add LICENSE to all org repos — decide product license explicitly; add MIT LICENSE file to ui (declared but missing) · created:2026-11-09
- T-045 Add SECURITY.md org-wide + enable GitHub private vulnerability reporting — health-tech handling patient data · created:2026-11-09
- T-046 Review + post upstream issue: TanStack/router duplicate-React in Cloudflare SSR (draft in handoff) · pri:low · created:2026-11-10 · #tanstack · #upstream
- T-047 Reshape WorkerSpec for true provider-agnosticism (spike findings) · created:2026-11-10 · #architecture
- T-084 Plugin lazy-loading: non-eager import.meta.glob for client manifests so disabled plugins code-split out of the browser bundle (P-058 phase-2 bundle cost) · pri:low · created:2026-11-11 · #plugins · links:P-058,P-060 · epic:platform
- T-085 Plugin-owned table DDL: today tables are host-owned in @ktb/db (cycle fix); if plugins should self-register schema again, needs a dep-free composition (e.g. codegen'd schema entry or registry pattern) · pri:low · created:2026-11-11 · #plugins · links:P-058 · epic:platform
