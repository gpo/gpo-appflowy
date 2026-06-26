# PR 01: Repository Scaffold and Upstream Subtrees

**Depends on:** PR 00 (docs, this branch)
**Blocks:** PR 02 through PR 08
**Estimate:** ~0.5 day

## Goal

Establish the monorepo skeleton, vendor the two upstream sources at their latest
non-RC release tags via `git subtree --squash`, and record the pinned versions in
`versions.yaml` as the human-readable source of truth.

Vendoring at a specific release tag (rather than tracking `main`) means:
- Builds are reproducible regardless of what upstream does after the tag.
- The sync workflow advances the pin deliberately, one release at a time.
- The repo is large (~1–2 GB) but self-contained.

## Scope

```
versions.yaml                 # pinned upstream release tags (appflowy_cloud, appflowy_flutter)
upstream/appflowy-cloud/      # git subtree snapshot at pinned release tag
upstream/appflowy-flutter/    # git subtree snapshot at pinned release tag
kubernetes/base/              # empty .gitkeep placeholders for each component
kubernetes/overlays/          # placeholders for prod, stage
kubernetes/argocd/            # placeholder
.gitignore
```

## Steps

1. Look up the latest non-RC release tag for each upstream repo (exclude tags
   matching `-rc`, `-alpha`, `-beta`, or `.rc`):
   - `https://github.com/AppFlowy-IO/AppFlowy-Cloud/releases`
   - `https://github.com/AppFlowy-IO/AppFlowy/releases`

2. Add the two subtrees, squash-merged at those tags:

   ```bash
   git subtree add --prefix upstream/appflowy-cloud \
     https://github.com/AppFlowy-IO/AppFlowy-Cloud.git <tag> --squash

   git subtree add --prefix upstream/appflowy-flutter \
     https://github.com/AppFlowy-IO/AppFlowy.git <tag> --squash
   ```

3. Create `versions.yaml` at the repo root, recording the pinned tags:

   ```yaml
   # Pinned upstream release versions. Updated weekly by sync-upstream.yml.
   # Both versions should be from the same release cycle (see carried verification #3).
   appflowy_cloud: vX.Y.Z
   appflowy_flutter: vX.Y.Z
   ```

4. Create the `kubernetes/` directory tree with `.gitkeep` files matching the layout
   in [02-architecture.md](../design/02-architecture.md):
   - `kubernetes/base/{appflowy-cloud,appflowy-worker,appflowy-search,admin-frontend,appflowy-web,gotrue,redis,cnpg,gateway}/.gitkeep`
   - `kubernetes/overlays/{prod,stage}/.gitkeep`
   - `kubernetes/argocd/.gitkeep`

5. Add `.gitignore` covering: kustomize build output (`/rendered/`, `*.rendered.yaml`),
   editor files (`.idea/`, `.vscode/`, `*.swp`), and rendered secrets.

## Acceptance Criteria

- `upstream/appflowy-cloud` and `upstream/appflowy-flutter` are present with their
  full source trees, vendored at non-RC release tags.
- `versions.yaml` records the same tags used for the subtree adds.
- The subtree commits record the upstream tag in their commit message.
- Both tags are from matching release cycles (same semver version, or explicitly noted
  if they differ); record this in the PR description.
- `kubernetes/` tree matches the documented layout.
- No secrets or rendered manifests committed.

## Notes

- The tags pinned here become the baseline for carried verification #3 (server/client
  compatibility). Record both version tags in the PR description.
- Do not modify upstream source in this PR. Patches, if ever needed, belong in
  dedicated PRs so subtree syncs stay clean.
- `appflowy_flutter` is the desktop/mobile client; it does not produce a container
  image. It is tracked for the version-compatibility audit trail and for any future
  patches to the client build.
- All five container images are built from `upstream/appflowy-cloud` only.
