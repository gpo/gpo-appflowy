# Infrastructure Prerequisites

Part of the [AppFlowy AGPL build design](./appflowy-agpl-build-design.md). These are one-time, mostly manual GCP and DNS tasks that must exist before any manifests run. They are tracked as a runbook in [PR 09: external setup runbook](../prs/pr-09-prereqs-and-secrets.md).

## 1. GCS Buckets

Create two buckets in GCP projects `gpo-eng-prod` and `gpo-eng-stage`:

```
gpo-appflowy-prod   (project: gpo-eng-prod,  region: northamerica-northeast2)
gpo-appflowy-stage  (project: gpo-eng-stage, region: northamerica-northeast2)
```

## 2. GCP Service Accounts

Create one service account per environment, following the `<resource>-<cluster>` naming convention:

```
appflowy-gpo-prod   (project: gpo-eng-prod)
appflowy-gpo-stage  (project: gpo-eng-stage)
```

Grant each service account `roles/storage.objectAdmin` on its respective bucket.

For GKE Workload Identity, bind each service account to the Kubernetes service account:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  appflowy-gpo-prod@gpo-eng-prod.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:gpo-eng-prod.svc.id.goog[appflowy/appflowy-cloud]"
```

Annotate the Kubernetes service account in the manifest:

```yaml
annotations:
  iam.gke.io/gcp-service-account: appflowy-gpo-prod@gpo-eng-prod.iam.gserviceaccount.com
```

## 3. pgvector Extension

AppFlowy requires the `vector` extension in Postgres. Enable it after the CNPG cluster is running:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

This can be declared in the CNPG cluster manifest using the `postgresql.parameters` and a bootstrap SQL hook, or applied manually once on first deploy.

## 4. Google Workspace OAuth Client

In Google Cloud Console for `gpo-eng-prod`:

1. APIs & Services > Credentials > Create OAuth 2.0 Client ID
2. Application type: Web application
3. Authorised redirect URIs:
   - `https://appflowy.gpotools.ca/gotrue/callback`
   - `https://appflowy.gpotoolsstaging.ca/gotrue/callback`
4. Note the client ID and secret. These go into the Kubernetes Secret for GoTrue.

## 5. Cloudflare DNS Records

Add DNS records in the `gpotools.ca` and `gpotoolsstaging.ca` zones pointing to the GKE Gateway external IP. Follow the existing pattern in `kubernetes/gateway/{prod,stage}/`.

## 6. Artifact Registry Repository

Verify the registry `northamerica-northeast2-docker.pkg.dev/<project>/gpo` exists and that the GitHub Actions service account has `roles/artifactregistry.writer`.
