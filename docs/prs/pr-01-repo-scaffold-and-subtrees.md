# PR 01: Repository Scaffold and Version Pins

**Depends on:** PR 00 (docs, this branch)
**Blocks:** PR 02 through PR 08
**Estimate:** ~0.5 day

## Goal

Establish the monorepo skeleton and pin the two upstream release versions so every
later PR has stable paths to build from and a clear source of truth for what
upstream revision is in use.

## Why not git subtree

The upstream repos are large (~1 GB each) and we only need specific release
snapshots, not rolling main. A `versions.yaml` file pins the two release tags; the
build workflow shallow-clones those tags at build time. This keeps the repo
lightweight, avoids subtree merge complexity, and makes the release pin explicit and
diff-friendly.

## Scope

```
versions.yaml                 # pinned upstream release tags
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

2. Create `versions.yaml` at the repo root:

   ```yaml
   # Pinned upstream release versions. Updated weekly by sync-upstream.yml.
   # Both versions should be from the same release cycle (see carried verification #3).
   appflowy_cloud: vX.Y.Z
   appflowy_flutter: vX.Y.Z
   ```

3. Create the `kubernetes/` directory tree with `.gitkeep` files matching the layout
   in [02-architecture.md](../design/02-architecture.md):
   - `kubernetes/base/{appflowy-cloud,appflowy-worker,appflowy-search,admin-frontend,appflowy-web,gotrue,redis,cnpg,gateway}/.gitkeep`
   - `kubernetes/overlays/{prod,stage}/.gitkeep`
   - `kubernetes/argocd/.gitkeep`

4. Add `.gitignore` covering: kustomize build output (`/rendered/`, `*.rendered.yaml`),
   editor files (`.idea/`, `.vscode/`, `*.swp`), and rendered secrets (`*secret*.yaml`
   not in `base/` or `overlays/`).

## Acceptance Criteria

- `versions.yaml` exists with non-RC release tags for both upstream repos.
- Both tags are from matching release cycles (same semver version, or explicitly noted
  if they differ); record this in the PR description.
- `kubernetes/` tree matches the documented layout.
- No secrets or rendered manifests committed.

## Notes

- The tags pinned here become the baseline for carried verification #3 (server/client
  compatibility). Record both version tags in the PR description.
- `appflowy_flutter` is the desktop/mobile client; it does not produce a container
  image. It is tracked here solely for the version-compatibility audit trail.
- All five container images are built from `appflowy_cloud` only.
