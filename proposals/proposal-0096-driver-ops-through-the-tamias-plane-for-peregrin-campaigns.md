+++
id = "P-096"
title = "driver ops through the tamias plane for peregrin campaigns"
status = "done"
created = "2026-11-18"
owner = "devin"
priority = "med"
labels = ["orchestration", "tamias"]
+++

## Context

Peregrin campaign nodes that need git/forge expertise currently shell out to CLIs (`lasius --root … vcs|forge … --json` + jq + exit-code emit maps + file-based state passing). The ktbhealth `workflows/pr-review` bundle (workspace PR #1) hit this directly. The right long-term shape is typed driver ops — but driver *responsibility* belongs to tamias (discovery, versioning, consent, the ndjson ops wire), not peregrin.

## Proposal

Expose installed `tamias-driver-<name>` ops through the tamias serve wire, and give peregrin one generic stdio-wire driver to consume them.

```
campaign yaml                tamias plane (owns ALL of this)
driver: ops            ──►    tamias-serve: driver.<name>.<op> dispatch
  driver: lasius              via existing discovery (PATH + drivers/)
  op: cli.forge.pr.merge      + consent + versioning (install-time)
  args: {...}                       │
                              lasius repo: ZERO changes —
                              tamias-driver-lasius already
                              serves forge.*/vcs.* ops
```

### Layer ownership

| layer | repo | change |
| --- | --- | --- |
| domain ops (`forge.*`/`vcs.*`) | lasius | none — `tamias-driver-lasius serve` already speaks the wire |
| driver-op dispatch (`driver.<name>.<query>`) | tamias | extend `tamias-serve` ops over existing driver discovery |
| wire client (`driver: ops` — spawn `serve`, invoke→op→event) | peregrin | one small generic driver; knows only the shared ndjson JSON-RPC wire (precedent: `peregrin-session` spawns `sepia serve`) |

### What campaigns get

`driver: ops` nodes with typed params and structured `${event.payload.*}` results — no shell, no jq, no `<worktree_base>/prs/<sha>` map files, no stderr text-matching for error classification. Works for every tamias driver (lasius, sturn, kuda, myco, bupha…), not just lasius.

### Alternatives considered

- **`peregrin-driver-lasius` crate in peregrin** — rejected: drivers are compiled into the binary (`Arc<dyn Driver>` registry, no plugin loading), and it puts domain expertise in the wrong repo.
- **Dynamic/.so drivers** — rejected: Rust ABI instability; the org already standardized on ndjson-JSON-RPC-over-stdio (`sepia serve`, `cuora serve`, `myco serve`, `tamias-serve`, every `tamias-driver-*`).
- **Status quo (shell + `lasius` shim)** — keeps working unchanged; this proposal is additive.

### Open questions

- Wire shape for driver-op dispatch: namespaced query (`driver.lasius.cli.forge.pr.merge`) vs `{driver, op, args}` envelope — pick whichever matches `tamias-serve`'s existing op grammar.
- Secrets crossing the wire: ops args are JSON; secret *names* only, resolved inside the driver's own secret source (same rule as everywhere).
- Long-running/ingress drivers (github-webhook style) stay in-process in peregrin — this proposal covers bounded worker ops only.
