# PR 07: CI, Upstream Sync and Image Builds

**Depends on:** PR 01 (subtrees exist)
**Blocks:** PR 08 (deploy chains off the build workflow)
**Estimate:** ~0.5 day

## Goal

Automate the weekly pull of upstream source into the subtrees and the parallel build/push of all five images to Artifact Registry.

## Scope

```
.github/workflows/sync-upstream.yml
.github/workflows/build-images.yml
```

## sync-upstream.yml

- Trigger: `schedule` cron `0 7 * * 0` (Sunday 07:00 UTC) plus `workflow_dispatch`.
- `permissions: contents: write`, checkout with `fetch-depth: 0`.
- Two steps run `git subtree pull --prefix upstream/appflowy-cloud ... main --squash` and the same for `upstream/appflowy-flutter`, then `git push origin main`.

Full YAML: [original design doc](../design/appflowy-agpl-build-design.md#1-sync-upstream-sync-upstreamyml).

## build-images.yml

- Trigger: `workflow_run` on completion of "Sync upstream AppFlowy sources" plus `workflow_dispatch`.
- Gate: `if: workflow_run.conclusion == 'success' || event_name == 'workflow_dispatch'`.
- `permissions: { contents: read, id-token: write }` for Workload Identity Federation.
- Matrix of five builds (name, context, dockerfile):

  | name | context | dockerfile |
  |---|---|---|
  | appflowy-cloud | `upstream/appflowy-cloud` | `.../Dockerfile` |
  | appflowy-worker | `upstream/appflowy-cloud` | `.../services/appflowy-worker/Dockerfile` |
  | appflowy-search | `upstream/appflowy-cloud` | `.../services/appflowy-search/Dockerfile` |
  | admin-frontend | `upstream/appflowy-cloud/admin_frontend` | `.../admin_frontend/Dockerfile` |
  | appflowy-web | `upstream/appflowy-cloud` | `.../services/appflowy-web/Dockerfile` |

- Auth via `google-github-actions/auth@v2`, then `gcloud auth configure-docker northamerica-northeast2-docker.pkg.dev`.
- `docker/build-push-action@v5` pushing `:${{ github.sha }}` and `:latest`, with `cache-from`/`cache-to: type=gha` scoped per image.

Full YAML: [original design doc](../design/appflowy-agpl-build-design.md#2-build-images-build-imagesyml).

## Required Secrets / Variables

| Secret | Purpose |
|---|---|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | WIF provider resource name |
| `GCP_ARTIFACT_REGISTRY_SA` | Service account with `roles/artifactregistry.writer` |

Verify the registry and the SA role per [PR 09](./pr-09-prereqs-and-secrets.md) before the first run.

## Acceptance Criteria

- `actionlint` passes on both workflows.
- A manual `workflow_dispatch` of build-images produces five images tagged with the commit SHA in Artifact Registry.
- The first cold `appflowy-cloud` build completes within the runner timeout (expect 20 to 40 min cold; raise `timeout-minutes` accordingly).
- Per-image GHA cache scopes are set so one image's eviction does not invalidate others.
