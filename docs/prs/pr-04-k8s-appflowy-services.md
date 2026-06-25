# PR 04: Kubernetes Base, AppFlowy Services

**Depends on:** PR 02 (Postgres, Redis), PR 03 (GoTrue)
**Blocks:** PR 05 (gateway backends), PR 06 (overlays)
**Estimate:** ~1 day

## Goal

Deploy the five built-from-source services: `appflowy-cloud`, `appflowy-worker`, `appflowy-search`, `admin-frontend`, and `appflowy-web`, including the Workload Identity service account that backs GCS access.

## Scope

```
kubernetes/base/appflowy-cloud/{deployment,service,serviceaccount,kustomization}.yaml
kubernetes/base/appflowy-worker/{deployment,kustomization}.yaml
kubernetes/base/appflowy-search/{deployment,service,kustomization}.yaml
kubernetes/base/admin-frontend/{deployment,service,kustomization}.yaml
kubernetes/base/appflowy-web/{deployment,service,kustomization}.yaml
```

Image names in base are bare (`appflowy-cloud`, etc.); the overlays rewrite them to the Artifact Registry path with a SHA tag ([PR 06](./pr-06-kustomize-overlays.md)).

## AppFlowy Cloud env (worker shares most of it)

```
APPFLOWY_ENVIRONMENT: production
APPFLOWY_DATABASE_URL: postgres://appflowy:<password>@appflowy-pg-rw.appflowy.svc:5432/appflowy
APPFLOWY_REDIS_URI: redis://:$(REDIS_PASSWORD)@redis.appflowy.svc:6379
APPFLOWY_GOTRUE_BASE_URL: http://gotrue.appflowy.svc:9999
APPFLOWY_BASE_URL: https://appflowy.gpotools.ca        # overlay-patched per env
APPFLOWY_S3_USE_MINIO: "false"
AWS_ACCESS_KEY: ""                                      # empty: Workload Identity provides creds
AWS_SECRET: ""
APPFLOWY_S3_BUCKET: gpo-appflowy-prod                   # overlay-patched per env
APPFLOWY_S3_REGION: northamerica-northeast2
AI_OPENAI_API_KEY: <openai-key>                         # from Secret; omit to disable AI
```

## Service Account (GCS via Workload Identity)

The `appflowy-cloud` ServiceAccount must carry:

```yaml
annotations:
  iam.gke.io/gcp-service-account: appflowy-gpo-prod@gpo-eng-prod.iam.gserviceaccount.com
```

The annotation value is environment-specific and is patched in the overlays. The GCP-side binding is set up in [PR 09](./pr-09-prereqs-and-secrets.md).

## Verification Task (carry into first deploy)

Confirm AppFlowy's AWS S3 client talks to GCS (`storage.googleapis.com`) with path-style or virtual-hosted URLs under Application Default Credentials. If it fails, drop a single-container MinIO gateway in front of GCS as a fallback. Tracked [Medium risk](../design/06-secrets-deployment-risks.md#risks).

## Acceptance Criteria

- All five components render via `kustomize build` and pass `kubeconform`.
- `appflowy-cloud` and `appflowy-worker` reference Postgres, Redis, and GoTrue by in-cluster DNS.
- The `appflowy-cloud` ServiceAccount exists and is wired for the Workload Identity annotation patch.
- No secrets committed.
