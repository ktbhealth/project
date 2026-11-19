+++
id = "I-002"
title = "peregrin serve: undeclared supersede defaults to constant key — global latest-wins drops queued triggers across shas"
status = "open"
created = "2026-11-19"
owner = "devin"
priority = "med"
+++

Found while investigating why pr-review dropped all check deliveries for ktbhealth/product PR #227 head c468dd18 (both CI batches, all marked *.superseded, merge gate never ran).

Mechanism: under 'peregrin campaign serve', supervisor::supervise runs every hosted campaign through RunSupervisor with 'Supersede::from(campaign.supersede.clone().unwrap_or_default())' (crates/peregrin-node/src/supervisor.rs ~line 565). A campaign yaml with NO 'supersede:' block therefore gets key="" (constant) + mode=latest-wins + cancel_running=false. Every trigger event — pull_request, check_run, check_suite, status, any sha — shares one queue key, so each new delivery evicts the pending job for ANY other sha while a run is active. Long agent-review runs (minutes) make this likely: whole check bursts collapse to whichever delivery landed last globally.

Evidence: ~/.local/share/peregrin/serve-history/pr-review-product.jsonl — all 40 events for c468dd18 are check_run.superseded/check_suite.superseded with payload key=""; zero reached ci.route; the surviving pending jobs were unrelated shas' deliveries.

Bundle-side fix shipped for ktbhealth pr-review: ktbhealth/workspace PR #6 declares 'supersede: {key: "{sha}", mode: latest-wins, cancel_running: false}' and bumps the bundle to 0.1.5.

Upstream follow-ups worth considering:
1. Same latent bug in 9aia/workspace workflows/pr-heal/main.yaml (identical 'No supersede' comment) — check whether it needs the same key.
2. Docs/default footgun: 'campaign serve' silently supervisor-izes campaigns without supersede and gives them global latest-wins. Either require an explicit supersede: for serve hosting, or default the key to something per-event (e.g. include event kind+sha) instead of constant ''.
3. Observability: displaced pending jobs only leave a *.superseded marker injected into the NEXT run for that key — under a constant key the markers are easy to misread as per-sha dedupe.
