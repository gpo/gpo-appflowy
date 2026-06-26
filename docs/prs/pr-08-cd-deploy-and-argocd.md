# PR 08: CD, Deploy Workflow and Argo CD Applications

**Depends on:** PR 06 (overlays), PR 07 (build workflow)
**Blocks:** First-time deploy ([PR 09](./pr-09-prereqs-and-secrets.md))
**Estimate:** ~0.5 day

## Goal

Close the GitOps loop: a deploy workflow rewrites overlay image tags and pushes to `main`, and two Argo CD `Application` resources sync the result into the prod and stage clusters.

## Scope

```
.github/workflows/deploy.yml
kubernetes/argocd/prod-application.yaml     # canonical copy lives in this repo
kubernetes/argocd/stage-application.yaml
```

Plus a companion change in **`gpo-platform-configs`** (separate PR in that repo): add the two Applications under `kubernetes/argocd-apps/{prod,stage}/application.yaml`.

## deploy.yml

- Trigger: `workflow_run` on completion of "Build and push AppFlowy images" plus `workflow_dispatch` with an optional `version` input.
- Gate on success or manual dispatch.
- Resolve the target version from `inputs.version` or, if absent, the `appflowy_cloud` value in `versions.yaml` (the same value the build workflow tagged the images with). The deploy keys off the upstream release version, not a commit SHA.
- For each of the five images, `sed -i` the matching `newTag:` line (identified by its trailing `# <image-name>` marker) in both overlays' `kustomization.yaml`.
- Commit (`--allow-empty`) and `git push origin main`.

Full YAML: [original design doc](../design/appflowy-agpl-build-design.md#3-deploy-deployyml). The `sed` marker convention must match byte-for-byte what [PR 06](./pr-06-kustomize-overlays.md) wrote; a mismatch silently rewrites nothing and the tags never advance. Dry-run the `sed` against the committed overlays before merge.

## Argo CD Application (per environment)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: appflowy
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/gpo/gpo-appflowy
    targetRevision: main
    path: kubernetes/overlays/prod      # stage: kubernetes/overlays/stage
  destination:
    server: https://kubernetes.default.svc
    namespace: appflowy
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

## Acceptance Criteria

- `actionlint` passes on deploy.yml.
- A dry-run of the workflow's `sed` against the committed overlays rewrites exactly the five `newTag:` lines (and nothing else) in both, to the version from `versions.yaml`.
- A `workflow_dispatch` run rewrites the five tags in both overlays and pushes a commit to `main`.
- After the platform-configs PR merges, Argo CD shows the `appflowy` Application `Synced` and `Healthy` in both environments.
- `selfHeal` and `prune` are enabled so drift is reconciled automatically.

## Sequencing Note

Land the `gpo-platform-configs` change only once base and overlays are merged and at least one image set exists, otherwise Argo CD will sync into a failing state. Coordinate with [PR 09](./pr-09-prereqs-and-secrets.md).
