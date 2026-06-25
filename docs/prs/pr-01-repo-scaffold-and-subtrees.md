# PR 01: Repository Scaffold and Upstream Subtrees

**Depends on:** PR 00 (docs, this branch)
**Blocks:** PR 02 through PR 08
**Estimate:** ~0.5 day

## Goal

Establish the monorepo skeleton and vendor the two upstream sources as `git subtree` prefixes, so every later PR has stable paths to build from and patch against.

## Scope

```
upstream/appflowy-cloud/      # subtree from AppFlowy-IO/AppFlowy-Cloud main
upstream/appflowy-flutter/    # subtree from AppFlowy-IO/AppFlowy main
kubernetes/base/              # empty .gitkeep placeholders for each component
kubernetes/overlays/          # placeholders for prod, stage
kubernetes/argocd/            # placeholder
.gitignore
```

## Steps

1. Add the two subtrees (pin the exact upstream commit in the commit message for reproducibility):

   ```bash
   git subtree add --prefix upstream/appflowy-cloud \
     https://github.com/AppFlowy-IO/AppFlowy-Cloud.git main --squash
   git subtree add --prefix upstream/appflowy-flutter \
     https://github.com/AppFlowy-IO/AppFlowy.git main --squash
   ```

2. Create the `kubernetes/` directory tree with `.gitkeep` files matching the layout in [02-architecture.md](../design/02-architecture.md).

3. Add a `.gitignore` for local kustomize build output, editor files, and any rendered secrets.

## Acceptance Criteria

- `upstream/appflowy-cloud` and `upstream/appflowy-flutter` are present with their full source trees.
- The subtree commits record the upstream SHA they were taken from.
- `kubernetes/` tree matches the documented layout.
- No secrets or rendered manifests committed.

## Notes

- The subtree SHAs pinned here become the baseline for the client/server compatibility pinning called out in the [risk register](../design/06-secrets-deployment-risks.md). Record both SHAs in the PR description.
- Do not modify upstream source in this PR. Patches, if ever needed, belong in dedicated PRs so upstream syncs stay clean.
