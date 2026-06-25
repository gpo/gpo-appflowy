# Secrets, First Deployment, Risks, and Effort

Part of the [AppFlowy AGPL build design](./appflowy-agpl-build-design.md).

## Secrets Management

Kubernetes Secrets are not committed to the repo. Use the existing External Secrets Operator pattern if present in `gpo-platform-configs`, otherwise create secrets manually on first deploy and document the keys.

The `appflowy-secrets` Secret in namespace `appflowy` must contain:

| Key | Description |
|---|---|
| `postgres-password` | CNPG appflowy user password |
| `redis-password` | Redis auth password |
| `gotrue-jwt-secret` | Random 256-bit string |
| `gotrue-google-client-id` | Google OAuth client ID |
| `gotrue-google-client-secret` | Google OAuth client secret |
| `openai-api-key` | OpenAI key for AI features and search embeddings |

## First-Time Deployment Sequence

Run these steps once before Argo CD takes over ongoing deploys. The full checklist is in [PR 09](../prs/pr-09-prereqs-and-secrets.md).

1. Create GCS buckets (prod and stage).
2. Create GCP service accounts and configure Workload Identity bindings.
3. Create Google OAuth client. Note client ID and secret.
4. Add DNS records in Cloudflare for `appflowy.gpotools.ca` and `appflowy.gpotoolsstaging.ca`.
5. Create the `gpo-appflowy` repo under the `gpo` org.
6. Add upstream subtrees.
7. Add all manifests and workflows.
8. Run `build-images` workflow manually to produce the first set of images.
9. Create Kubernetes secrets in both clusters.
10. Add Argo CD Application resources to `gpo-platform-configs`.
11. Trigger Argo CD sync and verify all pods reach `Running`.
12. Verify GoTrue at `https://appflowy.gpotools.ca/gotrue/health`.
13. Verify AppFlowy web and complete Google SSO with a `gpo.ca` account.
14. Run the weekly workflow manually to confirm the full sync, build, and deploy pipeline end to end.

## Risks

| Risk | Severity |
|---|---|
| AppFlowy relicenses or hollows out the AGPL repo | **High**, monitor commits; evaluate Outline plus Plane as fallback |
| Upstream breaks compatibility between server and clients | **Medium**, pin client and server to matching commits; fork both together |
| Flutter client source falls behind released binaries | **Medium**, already occurring; fork before gap widens further |
| GCS S3 compatibility with AppFlowy's AWS SDK client | **Medium**, verify on first deploy; MinIO gateway is the fallback |
| `GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS` support in AppFlowy's GoTrue fork | **Medium**, verify; fallback is application-layer domain check |
| Redis pod restart invalidates sessions | **Low**, mitigated by RDB persistence; users re-auth via Google SSO |
| GitHub Actions cache (10 GB) eviction causing slow Rust builds | **Low**, per-image cache scopes prevent cross-image eviction |
| Ongoing maintenance tracking a fast-moving upstream | **Medium**, budget ~2 to 4h/month |

## Effort Estimate

| Task | Estimate |
|---|---|
| Repo setup, subtrees, initial build | 1 day |
| Kubernetes manifests (base plus overlays) | 2 to 3 days |
| GitHub Actions workflows | 1 day |
| Argo CD integration into platform configs | 0.5 days |
| GCP prereqs (buckets, SA, OAuth, DNS) | 0.5 days |
| First-time deploy and smoke testing | 1 day |
| Apple Developer account (iOS contingency) | $99 USD/year |
| Ongoing maintenance | ~2 to 4h/month |
