# Kubernetes Manifests

Part of the [AppFlowy AGPL build design](./appflowy-agpl-build-design.md). The canonical, full manifests live in the [original design doc](./appflowy-agpl-build-design.md#kubernetes-manifests). Ready-to-apply versions are extracted into the per-PR task specs linked below.

All manifests use Kustomize. Base manifests are environment-agnostic; overlays patch image tags and environment-specific values.

## Components and Where They Are Built

| Component | Base path | Built in PR | Key design notes |
|---|---|---|---|
| CNPG cluster | `kubernetes/base/cnpg/` | [PR 02](../prs/pr-02-k8s-data-layer.md) | Single instance, 8Gi storage, `shared_preload_libraries: vector`, `postInitSQL` creates the `vector` extension |
| Redis | `kubernetes/base/redis/` | [PR 02](../prs/pr-02-k8s-data-layer.md) | RDB persistence (`save 900 1 300 10`, `appendonly no`), password from Secret, 1Gi PVC |
| GoTrue | `kubernetes/base/gotrue/` | [PR 03](../prs/pr-03-k8s-gotrue.md) | Google SSO only, email signup disabled, domain restricted to `gpo.ca` |
| AppFlowy Cloud | `kubernetes/base/appflowy-cloud/` | [PR 04](../prs/pr-04-k8s-appflowy-services.md) | GCS via S3 client and Workload Identity, optional OpenAI key |
| AppFlowy Worker | `kubernetes/base/appflowy-worker/` | [PR 04](../prs/pr-04-k8s-appflowy-services.md) | Shares config with cloud |
| AppFlowy Search | `kubernetes/base/appflowy-search/` | [PR 04](../prs/pr-04-k8s-appflowy-services.md) | pgvector-backed embeddings |
| Admin Frontend | `kubernetes/base/admin-frontend/` | [PR 04](../prs/pr-04-k8s-appflowy-services.md) | Admin UI |
| AppFlowy Web | `kubernetes/base/appflowy-web/` | [PR 04](../prs/pr-04-k8s-appflowy-services.md) | Primary web client, default route target |
| Gateway / HTTPRoute | `kubernetes/base/gateway/` | [PR 05](../prs/pr-05-k8s-gateway.md) | Path-based routing for `/api`, `/ws`, `/gotrue`, and `/` |
| Overlays (prod, stage) | `kubernetes/overlays/{prod,stage}/` | [PR 06](../prs/pr-06-kustomize-overlays.md) | Image tag substitution and env-specific patches |

## Cross-Cutting Design Notes

### GCS storage via the AWS S3 client

AppFlowy's S3 client uses the AWS SDK. To use GCS with Workload Identity, set `APPFLOWY_S3_USE_MINIO: "false"`, leave the access key and secret empty, and configure the GKE service account annotation. The AWS SDK respects Application Default Credentials when no key is provided, and GCS exposes an S3-compatible endpoint at `storage.googleapis.com`.

> **Verify on first deploy:** Confirm AppFlowy's S3 client supports GCS path-style or virtual-hosted URLs. If not, deploy a lightweight MinIO gateway in front of GCS as a fallback. This is a single-container proxy, not a storage system.

### GoTrue domain restriction

`GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS: "gpo.ca"` restricts Google SSO to accounts from `gpo.ca` only. Verify this env var is supported in AppFlowy's GoTrue fork; if not, domain restriction can be enforced at the AppFlowy application layer or via a GoTrue hook.

### Gateway routing

Follow the existing pattern in `gpo-platform-configs/kubernetes/gateway/`. Create an `HTTPRoute` routing all traffic for `appflowy.gpotools.ca` to the `appflowy-web` service, with path-based routing for `/api`, `/ws`, and `/gotrue` sub-paths forwarding to their respective services. The `parentRefs` gateway name and namespace must match the existing gateway in `gpo-platform-configs`.
