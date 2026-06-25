# CLAUDE.md

Guidance for working in this repository.

## What This Repo Is

`gpo-appflowy` self-hosts [AppFlowy](https://github.com/AppFlowy-IO/AppFlowy-Cloud) on GKE, built entirely from AGPL-3.0 source to avoid the closed commercial images (which impose a `max_users: 1` limit that does not exist in the open source). It is a monorepo holding Kubernetes manifests, version pins for the two upstream repos, and the CI/CD that builds images and drives GitOps deploys.

The full plan lives in [`docs/`](./docs/README.md). Read the [roadmap](./docs/roadmap.md) before starting work; it breaks the design into per-PR task specs under [`docs/prs/`](./docs/prs/).

## Layout

```
versions.yaml        # pinned upstream release tags (appflowy_cloud, appflowy_flutter)
kubernetes/
  base/              # env-agnostic Kustomize bases, one dir per component
  overlays/          # prod and stage; image tags + env-specific patches
  argocd/            # Argo CD Application resources (canonical copies)
.github/workflows/   # sync-upstream, build-images, deploy
docs/                # design (split + original) and per-PR task specs
```

## How the Pipeline Fits Together

Weekly, in sequence (Sunday 07:00 UTC):

1. `sync-upstream.yml` checks for new non-RC releases on both upstream repos and updates `versions.yaml` if newer tags are available, then pushes to `main`.
2. `build-images.yml` shallow-clones the pinned release tags, builds the five images, and pushes them to Artifact Registry tagged with the upstream release version.
3. `deploy.yml` rewrites the `newTag:` lines in both overlays (using the version from `versions.yaml`) and pushes to `main`.
4. Argo CD (`selfHeal` + `prune`) reconciles both clusters from `kubernetes/overlays/{prod,stage}`.

The five built-from-source images: `appflowy-cloud`, `appflowy-worker`, `appflowy-search`, `admin-frontend`, and `appflowy-web`. GoTrue uses the open upstream image. Postgres is CloudNativePG, Redis runs in-cluster, and object storage is GCS via the AWS S3 client and Workload Identity.

## Conventions

- **Version pins are the source of truth.** Upstream release versions live in `versions.yaml`. To pin a different release, update that file and let the pipeline rebuild. Do not commit upstream source into the repo.
- **One PR per spec.** Follow the breakdown in [`docs/roadmap.md`](./docs/roadmap.md); keep the manifest track as separate PRs, not one mega-PR.
- **No secrets in git.** All sensitive values flow through the `appflowy-secrets` Secret or External Secrets Operator. Never commit rendered manifests containing secrets.
- **Validate before review:** `kustomize build` plus `kubeconform` for manifest changes, `actionlint` for workflow changes.
- **Environment values live in overlays**, not bases. Bases stay env-agnostic.

## Out-of-Repo Coupling

`gpo-platform-configs` holds the live Argo CD Applications, the Gateway, the CNPG operator, and any ESO config. Manifests here assume those exist; confirm before first deploy. The Argo CD Applications are mirrored in `kubernetes/argocd/` for reference.

## Carried Verifications (close before go-live)

1. GCS works with AppFlowy's AWS S3 client (fallback: MinIO gateway).
2. `GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS` restricts SSO to `gpo.ca` (fallback: app-layer check).
3. Server and Flutter client are pinned to compatible upstream commits.
