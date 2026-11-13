+++
id = "P-068"
title = "OCI IaC for LGTM host: infra/oci OpenTofu adapter behind infra-ports"
status = "done"
created = "2026-11-13"
priority = "med"
labels = ["architecture-platform", "infra", "observability"]
links = ["T-109", "P-066"]
+++

## Question

Can Alchemy (or Pulumi) provision the OCI VM for LGTM instead of manual console steps? Would it be consistent?

## Findings

- **Alchemy cannot do OCI.** Alchemy 2.0-beta is Cloudflare-scoped — installed providers are @alchemy.run/{cloudflare,docker,acme,sql}. No OCI provider exists; alchemy.run is a CF-first tool.
- **Pulumi has no native OCI provider.** Only path is bridging the community `oracle/oci` Terraform provider via pulumi-tfgen — immature, and introduces a SECOND IaC tool + state backend into a repo that already standardizes on Alchemy. Inconsistent.
- **Terraform/OpenTofu is the standard for OCI** — Oracle maintains the official `oracle/oci` provider; Always-Free shapes (VM.Standard.A1.Flex) are well supported.

## The consistent answer

The repo ALREADY has the ports/adapters infra pattern:
- `infra/ports` (`@ktb/infra-ports`): DeployWorker / DeployStaticSite / DeploySqlDatabase Effect ports
- `infra/cloudflare`: Alchemy-backed adapter

So the consistent move is NOT a new tool — it's a new port + adapter:

1. `infra/ports`: add `DeployHost`/`DeployVm` port (spec: name, image/OS, shape, sshKeys, cloudInit, region; returns: host handle/IP).
2. `infra/oci`: OpenTofu-based adapter — `infra/oci/*.tf` (VCN/subnet/security-list/instance, optional public IP, free-tier shape pinned) invoked via `tofu apply` from an Effect wrapper, or committed plan+apply scripts.
3. `infra/lgtm` stays pure runtime config (compose/cloudflared/provisioning) — host-agnostic.

## Recommendation

- **Now (T-109)**: manual OCI console per infra/lgtm/README.md — unblocks today, one VM.
- **Follow-up (this proposal)**: implement `DeployHost` port + `infra/oci` OpenTofu adapter when a second host is needed (staging LGTM, DR rebuild, or org domain arrives and infra grows). Premature for a single VM but the right shape when it does.

Alternative if we want IaC sooner: a plain `infra/lgtm/terraform/` or `infra/oci/` OpenTofu module without the ports abstraction — simpler, still consistent tooling-wise (OCI's official provider), upgrade to port/adapter when a second consumer appears.
