# Documentation

Self-hosting AppFlowy on GKE from AGPL source, replacing the closed commercial images. Start with the [roadmap](./roadmap.md) for the execution plan, or the [design overview](./design/00-overview.md) for the rationale.

## Design

The design is split into focused sections. The full, original document is preserved verbatim at [`design/appflowy-agpl-build-design.md`](./design/appflowy-agpl-build-design.md).

| Section | Covers |
|---|---|
| [00 Overview](./design/00-overview.md) | Core finding, the two-binary problem, what we keep and lose |
| [01 Licensing and clients](./design/01-licensing-and-clients.md) | CLA/relicensing risk, upstream trajectory, client app contingency |
| [02 Architecture](./design/02-architecture.md) | Monorepo layout, subtrees, Redis usage and persistence |
| [03 Infrastructure prerequisites](./design/03-infrastructure-prerequisites.md) | GCS, service accounts, pgvector, OAuth, DNS, registry |
| [04 Kubernetes manifests](./design/04-kubernetes-manifests.md) | Component map and cross-cutting design notes |
| [05 CI/CD and Argo CD](./design/05-cicd-and-argocd.md) | Weekly sync, build, deploy pipeline and GitOps |
| [06 Secrets, deploy, risks](./design/06-secrets-deployment-risks.md) | Secret keys, first-deploy sequence, risk register, effort |

## Execution

| Doc | Purpose |
|---|---|
| [Roadmap](./roadmap.md) | PR breakdown, dependency graph, parallelization, conventions |
| [`prs/`](./prs/) | One task spec per PR, executable without re-reading the full design |

## Repo Guide

See [`/CLAUDE.md`](../CLAUDE.md) at the repo root for conventions, layout, and how the GitOps pipeline fits together.
