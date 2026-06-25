# CI/CD and Argo CD

Part of the [AppFlowy AGPL build design](./appflowy-agpl-build-design.md). Full workflow YAML lives in the [original design doc](./appflowy-agpl-build-design.md#github-actions-workflows) and the per-PR specs ([PR 07](../prs/pr-07-ci-sync-and-build.md), [PR 08](../prs/pr-08-cd-deploy-and-argocd.md)).

## Weekly Pipeline

All three workflows run in sequence on Sunday at 07:00 UTC (03:00 ET):

```
sync-upstream (07:00 UTC)
  → on success: build-images (triggered by workflow_run)
    → on success: deploy (triggered by workflow_run)
```

## 1. Sync Upstream (`sync-upstream.yml`)

Pulls upstream changes into the two `git subtree` prefixes (`upstream/appflowy-cloud`, `upstream/appflowy-flutter`) with `--squash`, then pushes to `main`. Built in [PR 07](../prs/pr-07-ci-sync-and-build.md).

## 2. Build Images (`build-images.yml`)

Triggered on `workflow_run` completion of the sync workflow (or `workflow_dispatch`). Uses a matrix to build and push all five images in parallel to Artifact Registry, authenticating via Workload Identity Federation. Built in [PR 07](../prs/pr-07-ci-sync-and-build.md).

The five matrix targets: `appflowy-cloud`, `appflowy-worker`, `appflowy-search`, `admin-frontend`, and `appflowy-web`.

> **Build time:** The `appflowy-cloud` Rust build is the bottleneck at 20 to 40 min on a cold cache. GitHub-hosted runners get 10 GB of Actions cache. With `cargo-chef` caching in the Dockerfile and `cache-from: type=gha`, subsequent weekly builds should drop to 5 to 10 min per image. Per-image cache scopes prevent single-image eviction from affecting others.

## 3. Deploy (`deploy.yml`)

Triggered on `workflow_run` completion of build-images (or `workflow_dispatch` with an optional SHA input). Rewrites the image tags in both overlays' `kustomization.yaml`, commits, and pushes to `main`. Built in [PR 08](../prs/pr-08-cd-deploy-and-argocd.md).

## Argo CD Applications

Two `Application` resources, one per environment, point Argo CD at `kubernetes/overlays/{prod,stage}` in this repo. They are added to `gpo-platform-configs/kubernetes/argocd-apps/{prod,stage}/application.yaml` following the existing pattern. Built in [PR 08](../prs/pr-08-cd-deploy-and-argocd.md).

`automated.selfHeal: true` means Argo CD re-applies on any drift. The weekly deploy workflow commits updated image tags to `main`, and Argo CD picks them up automatically, with no manual promotion step.
