# PR 09: Prerequisites, Secrets, and First-Time Deploy Runbook

**Depends on:** PR 02 through PR 08 (the things being deployed)
**Type:** Mostly operational (GCP, DNS, kubectl, Argo CD), plus optional ESO manifests
**Estimate:** ~1 day, plus 0.5 day of GCP/DNS prep

## Goal

Carry out the one-time, mostly manual setup that the manifests assume, create the cluster secrets, and run the first deployment end to end. This PR is primarily a runbook plus any External Secrets Operator (ESO) manifests if that pattern exists in `gpo-platform-configs`.

## Manual Prerequisites

See [03-infrastructure-prerequisites.md](../design/03-infrastructure-prerequisites.md) for full commands. Checklist:

- [ ] GCS buckets `gpo-appflowy-prod` and `gpo-appflowy-stage` (region `northamerica-northeast2`).
- [ ] Service accounts `appflowy-gpo-prod` and `appflowy-gpo-stage`, each granted `roles/storage.objectAdmin` on its bucket.
- [ ] Workload Identity bindings from each GCP SA to `serviceAccount:<project>.svc.id.goog[appflowy/appflowy-cloud]`.
- [ ] Google OAuth 2.0 Web client with the two redirect URIs; record client ID and secret.
- [ ] Cloudflare DNS for `appflowy.gpotools.ca` and `appflowy.gpotoolsstaging.ca` to the Gateway IP.
- [ ] Artifact Registry `northamerica-northeast2-docker.pkg.dev/<project>/gpo` exists; GitHub Actions SA has `roles/artifactregistry.writer`.

## Secrets

Decide between ESO (preferred if `gpo-platform-configs` already uses it) and manual `kubectl create secret`. The `appflowy-secrets` Secret in namespace `appflowy` must hold: `postgres-password`, `redis-password`, `gotrue-jwt-secret`, `gotrue-google-client-id`, `gotrue-google-client-secret`, `openai-api-key`. See [06-secrets-deployment-risks.md](../design/06-secrets-deployment-risks.md#secrets-management).

If manual:

```bash
kubectl create namespace appflowy --context gpo-prod
kubectl create secret generic appflowy-secrets -n appflowy \
  --from-literal=postgres-password=<...> \
  --from-literal=redis-password=<...> \
  --from-literal=gotrue-jwt-secret=<...> \
  --from-literal=gotrue-google-client-id=<...> \
  --from-literal=gotrue-google-client-secret=<...> \
  --from-literal=openai-api-key=<...> \
  --context gpo-prod
# repeat for gpo-stage
```

## First-Time Deploy and Smoke Test

1. Manually run `build-images` (`workflow_dispatch`) to produce the first image set.
2. Create the secrets in both clusters.
3. Merge the `gpo-platform-configs` Argo CD Applications.
4. Trigger Argo CD sync; verify all pods reach `Running`.
5. `GET https://appflowy.gpotools.ca/gotrue/health`.
6. Load `https://appflowy.gpotools.ca` and complete Google SSO with a `gpo.ca` account.
7. Verify these tracked items resolve:
   - GCS S3 compatibility (else stand up the MinIO gateway fallback).
   - `GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS` actually restricts to `gpo.ca`.
8. Run the full weekly pipeline once manually (sync, build, deploy) to confirm it works end to end.

## Acceptance Criteria

- All prerequisite checkboxes complete.
- Both environments reach `Synced` and `Healthy` in Argo CD with all pods `Running`.
- Google SSO works with a `gpo.ca` account and is rejected for non-`gpo.ca` accounts.
- File upload (GCS path) verified.
