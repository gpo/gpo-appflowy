# PR 06: Kustomize Overlays, Prod and Stage

**Depends on:** PR 02 through PR 05 (all base components)
**Blocks:** PR 08 (Argo CD points at these overlay paths)
**Estimate:** ~0.5 day

## Goal

Provide `prod` and `stage` overlays that pull in the base, rewrite image names to the Artifact Registry path with a deployable SHA tag, and patch all environment-specific values.

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
    newTag: <SHA>   # rewritten by the deploy workflow
  - name: appflowy-worker
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/appflowy-worker
    newTag: <SHA>
  - name: appflowy-search
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/appflowy-search
    newTag: <SHA>
  - name: admin-frontend
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/admin-frontend
    newTag: <SHA>
  - name: appflowy-web
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/appflowy-web
    newTag: <SHA>
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

The deploy workflow ([PR 08](./pr-08-cd-deploy-and-argocd.md)) rewrites the five `newTag:` lines. Keep each `newTag` line individually addressable by the workflow's `sed` (a trailing `# <image-name>` marker comment is the agreed convention; keep base and overlay marker format identical across both overlays).

## Acceptance Criteria

- `kustomize build kubernetes/overlays/prod` and `.../stage` both render fully and pass `kubeconform`.
- Image names resolve to the correct per-environment registry path.
- Every environment-specific value differs correctly between prod and stage.
- The `newTag` marker format matches what the deploy workflow rewrites.
