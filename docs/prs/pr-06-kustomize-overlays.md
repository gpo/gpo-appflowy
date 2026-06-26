# PR 06: Kustomize Overlays, Prod and Stage

**Depends on:** PR 02 through PR 05 (all base components)
**Blocks:** PR 08 (Argo CD points at these overlay paths)
**Estimate:** ~0.5 day

## Goal

Provide `prod` and `stage` overlays that pull in the base, rewrite image names to the Artifact Registry path with the pinned upstream release version as the tag, and patch all environment-specific values.

## Scope

```
kubernetes/overlays/prod/kustomization.yaml
kubernetes/overlays/prod/gotrue-patch.yaml
kubernetes/overlays/prod/appflowy-cloud-patch.yaml
kubernetes/overlays/prod/serviceaccount-patch.yaml
kubernetes/overlays/stage/...   # same set, stage values
```

## Prod kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: appflowy
resources:
  - ../../base
images:
  - name: appflowy-cloud
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/appflowy-cloud
    newTag: v0.0.0   # appflowy-cloud  (rewritten by deploy workflow)
  - name: appflowy-worker
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/appflowy-worker
    newTag: v0.0.0   # appflowy-worker
  - name: appflowy-search
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/appflowy-search
    newTag: v0.0.0   # appflowy-search
  - name: admin-frontend
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/admin-frontend
    newTag: v0.0.0   # admin-frontend
  - name: appflowy-web
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/appflowy-web
    newTag: v0.0.0   # appflowy-web
patches:
  - path: gotrue-patch.yaml
  - path: appflowy-cloud-patch.yaml
  - path: serviceaccount-patch.yaml
```

## Per-environment patch values

| Value | prod | stage |
|---|---|---|
| Host | `appflowy.gpotools.ca` | `appflowy.gpotoolsstaging.ca` |
| GCS bucket | `gpo-appflowy-prod` | `gpo-appflowy-stage` |
| GCP project in registry path | `gpo-eng-prod` | `gpo-eng-stage` |
| Workload Identity SA | `appflowy-gpo-prod@gpo-eng-prod...` | `appflowy-gpo-stage@gpo-eng-stage...` |
| GoTrue site URL / redirect | `https://appflowy.gpotools.ca/...` | `https://appflowy.gpotoolsstaging.ca/...` |

## Deploy-workflow contract

The deploy workflow ([PR 08](./pr-08-cd-deploy-and-argocd.md)) rewrites the five `newTag:` lines to the `appflowy_cloud` version read from `versions.yaml`. Each `newTag` line carries a trailing `# <image-name>` marker comment so the workflow's `sed` can target it individually; this marker is a contract between the two PRs and must match byte-for-byte what the deploy workflow's pattern expects. Keep the marker format identical across both overlays.

Validate the contract, do not just eyeball it: run the deploy workflow's exact `sed` against the committed overlays and confirm it rewrites all five `newTag:` lines (and nothing else) to the target version. A marker mismatch is a silent failure: `sed` rewrites nothing, the tags never advance, and Argo CD keeps deploying the old image with no error surfaced anywhere.

## Acceptance Criteria

- `kustomize build kubernetes/overlays/prod` and `.../stage` both render fully and pass `kubeconform`.
- Image names resolve to the correct per-environment registry path.
- Every environment-specific value differs correctly between prod and stage.
- Each `newTag:` line carries its `# <image-name>` marker, and a dry-run of the deploy workflow's `sed` rewrites exactly those five lines (and nothing else) in both overlays.
