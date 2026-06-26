# PR 07: CI, Upstream Sync and Image Builds

**Depends on:** PR 01 (subtrees and versions.yaml exist)
**Blocks:** PR 08 (deploy chains off the build workflow)
**Estimate:** ~0.5 day

## Goal

Automate the weekly check for new upstream releases, advance the subtree pins and
`versions.yaml` when a new non-RC release is found, and build/push all five images
to Artifact Registry tagged with the upstream release version.

## Scope

```
.github/workflows/sync-upstream.yml
.github/workflows/build-images.yml
```

## sync-upstream.yml

- Trigger: `schedule` cron `0 7 * * 0` (Sunday 07:00 UTC) plus `workflow_dispatch`.
- `permissions: contents: write`, checkout with `fetch-depth: 0` (required for subtree
  operations).
- Logic for each of the two repos (`AppFlowy-Cloud`, `AppFlowy`):
  1. Fetch the latest release tag via the GitHub API (`/repos/{owner}/{repo}/releases/latest`).
  2. Reject any tag matching `-rc`, `-alpha`, `-beta`, or `.rc` (re-query by listing
     releases and skipping pre-releases if the `/latest` endpoint returns one).
  3. Compare to the current value in `versions.yaml`.
  4. If newer, run `git subtree pull --prefix upstream/<name> <url> <new-tag> --squash`
     and update the corresponding key in `versions.yaml`.
- If either version changed, commit both the subtree merge and the `versions.yaml`
  update together with a message like
  `chore: bump appflowy-cloud to vX.Y.Z, appflowy-flutter to vX.Y.Z`, then push to `main`.
- If nothing changed, exit without a commit.

Requires `GITHUB_TOKEN` (default token with `contents: write` is sufficient).

## build-images.yml

- Trigger: `workflow_run` on completion of "Sync upstream AppFlowy sources" plus `workflow_dispatch`.
- Gate: `if: github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch'`.
- `permissions: { contents: read, id-token: write }` for Workload Identity Federation.
- Read `appflowy_cloud` version from `versions.yaml` at the start of the job (used as
  the image tag).
- Matrix of five builds from the vendored source. Verify each Dockerfile path exists
  in the real upstream tree at the pinned tag before shipping (see gotcha note below):

  | name | context | dockerfile |
  |---|---|---|
  | appflowy-cloud | `upstream/appflowy-cloud` | `Dockerfile` |
  | appflowy-worker | `upstream/appflowy-cloud` | `services/appflowy-worker/Dockerfile` |
  | appflowy-search | `upstream/appflowy-cloud` | `services/appflowy-search/Dockerfile` |
  | admin-frontend | `upstream/appflowy-cloud` | `admin_frontend/Dockerfile` |
  | appflowy-web | `upstream/appflowy-cloud` | `services/appflowy-web/Dockerfile` |

- Auth via `google-github-actions/auth@v2`, then `gcloud auth configure-docker northamerica-northeast2-docker.pkg.dev`.
- `docker/build-push-action@v5` pushing `:<upstream-version>` and `:latest`, with
  `cache-from`/`cache-to: type=gha` scoped per image name.
- Image tag is the upstream release version (e.g., `v0.9.1`), not `github.sha`. The
  deploy workflow reads the same version from `versions.yaml` when updating overlay
  `newTag:` values.

**Gotcha:** Dockerfile paths above are from the spec and may drift between releases.
After PR 01 lands, verify each path exists in the real upstream tree at the pinned
tag. Fix the matrix to match reality; do not ship paths that don't exist.

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
