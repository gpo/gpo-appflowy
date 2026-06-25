# Architecture: Repository Structure and Redis

Part of the [AppFlowy AGPL build design](./appflowy-agpl-build-design.md).

## Repository Structure

A single monorepo (`gpo/gpo-appflowy`) contains everything: upstream source via `git subtree`, GitHub Actions workflows, and Kubernetes manifests. Argo CD is pointed at the `kubernetes/` directory within this repo, which is valid; an Argo CD `Application` resource simply takes a `repoURL` and `path`.

```
gpo-appflowy/
├── upstream/
│   ├── appflowy-cloud/          # git subtree from AppFlowy-IO/AppFlowy-Cloud
│   └── appflowy-flutter/        # git subtree from AppFlowy-IO/AppFlowy
├── kubernetes/
│   ├── base/                    # Kustomize base manifests (env-agnostic)
│   │   ├── appflowy-cloud/
│   │   ├── appflowy-worker/
│   │   ├── appflowy-search/
│   │   ├── admin-frontend/
│   │   ├── appflowy-web/
│   │   ├── gotrue/
│   │   ├── redis/
│   │   ├── cnpg/
│   │   └── gateway/
│   ├── overlays/
│   │   ├── prod/                # Kustomize overlay: prod image tags, replicas, secrets refs
│   │   └── stage/              # Kustomize overlay: stage image tags
│   └── argocd/
│       ├── prod-application.yaml
│       └── stage-application.yaml
└── .github/
    └── workflows/
        ├── sync-upstream.yml    # Weekly: pull upstream changes into subtrees
        ├── build-images.yml     # Weekly: build and push images to Artifact Registry
        └── deploy.yml           # Weekly: update image tags in overlays, commit, trigger Argo CD sync
```

The Argo CD `Application` resources in `kubernetes/argocd/` are added to `gpo-platform-configs` under `kubernetes/argocd-apps/{prod,stage}/application.yaml` alongside the existing apps.

## Redis Usage

AppFlowy uses Redis for three purposes:

1. **Session storage** (`RedisSessionStore`), stateful; Redis restart invalidates all active sessions, requiring users to re-authenticate via Google SSO.
2. **Redis Streams** (`collab_stream`), real-time collaboration event routing between server instances; messages are persistent until trimmed.
3. **Awareness gossip**, cursor/presence sync; ephemeral pub/sub.

**Implication:** Deploy Redis with RDB persistence (`save 900 1 300 10`, `appendonly no`). This survives pod restarts without mass session invalidation. Full AOF durability is not required since session loss is recoverable.
