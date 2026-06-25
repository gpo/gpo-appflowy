# Design: AppFlowy AGPL Deployment on GKE

**Status:** Draft  
**Upstream:** https://github.com/AppFlowy-IO/AppFlowy-Cloud (AGPL-3.0)  
**Monorepo:** `github.com/gpo/gpo-appflowy` (to be created)  
**Platform config:** `github.com/gpo/gpo-platform-configs` (existing)

---

## The Core Finding

The user limit is not in the AGPL source. Direct inspection of `src/biz/user/user_verify.rs`, `src/biz/workspace/ops.rs`, and a full `src/` grep for `max_users`, `license`, `plan_limit`, and related terms found nothing. The `"Free plan limits - max_users: 1"` message in community bug reports comes entirely from the closed Docker Hub binary, not this codebase.

Building from source and substituting your own images for the four closed ones is the complete solution.

---

## The Two-Binary Problem

`docker-compose.yml` pulls a mix of open and closed images:

| Image | Source | Action |
|---|---|---|
| `appflowyinc/appflowy_cloud` | Closed commercial fork | Build from AGPL source |
| `appflowyinc/admin_frontend` | Closed commercial fork | Build from AGPL source |
| `appflowyinc/appflowy_worker` | Closed commercial fork | Build from AGPL source |
| `appflowyinc/appflowy_search` | Closed commercial fork | Build from AGPL source |
| `appflowyinc/appflowy_web` | AGPL (recently re-opened) | Build from AGPL source |
| `appflowyinc/gotrue` | Open (Supabase Auth fork) | Use upstream image |
| `minio/minio`, `pgvector/pgvector`, `redis` | Fully open | Not used -- replaced by GCS, CNPG, in-cluster Redis |

---

## What You Lose Without the Commercial Binary

**You lose:**
- **AI response quotas managed by Stripe** -- the AGPL source has no AI billing enforcement; supply an OpenAI API key directly via env vars and it works without limits
- **Stripe billing and subscription management UI** -- irrelevant for internal use
- **Priority support** -- the Team/Enterprise SLA

**You keep everything GPO actually needs:**
- Unlimited users and workspaces
- Real-time collaborative editing
- Databases (grid, kanban, calendar)
- Wiki and documents
- Semantic search via `appflowy_search` (uses pgvector, not Elasticsearch)
- Google Workspace SSO
- GCS file storage
- Full mobile and desktop client support

---

## Licence Situation and Long-Term Risk

### Can AppFlowy change the licence?

Yes. AppFlowy requires all contributors to sign a CLA, granting them the right to relicense contributions -- standard open-core practice. However, all source and releases prior to any licence change remain permanently available under the original AGPL. Any version forked today stays AGPL forever. A relicensing would only affect new upstream commits.

### Current trajectory

A GitHub issue raised in February 2026 noted that released binaries cannot be reproduced from the public AGPL source. AppFlowy confirmed they are operating a closed commercial fork alongside the AGPL repo, without disputing the compliance concern. The Flutter client repo exhibits the same pattern: source currently at `v0.11.4` while release tags run to `v0.12.5`. Development is consolidating into private repos with periodic public merges.

This is the standard path toward either a full relicensing or an increasingly hollow public repo. If AppFlowy relicenses, the last clean AGPL commit becomes the pinned version. Falling off main after 12-18 months would be noticeable given the development pace. **Evaluate Outline + Plane as a fallback hedge.**

---

## Client Apps

The Flutter client (`AppFlowy-IO/AppFlowy`, AGPL) covers iOS, Android, desktop, and web from a single codebase, with CI workflows for both mobile platforms.

### Contingency: Publishing Your Own Build

If official apps become incompatible with a pinned server version:

- **Android:** Build APK/AAB from source, sign with GPO's key, distribute via Play Store or sideload.
- **iOS:** Requires Apple Developer account ($99 USD/year). Distribute via TestFlight (90-day expiry, 10k user cap -- adequate for volunteers) or App Store under GPO's bundle ID.

**The hedge:** Fork the Flutter client at the same commit as the server pin. AGPL requires publishing modifications only if distributing publicly; distributing to GPO volunteers does not trigger this.

---

## Redis Usage

AppFlowy uses Redis for three purposes:

1. **Session storage** (`RedisSessionStore`) -- stateful; Redis restart invalidates all active sessions, requiring users to re-authenticate via Google SSO
2. **Redis Streams** (`collab_stream`) -- real-time collaboration event routing between server instances; messages are persistent until trimmed
3. **Awareness gossip** -- cursor/presence sync; ephemeral pub/sub

**Implication:** Deploy Redis with RDB persistence (`save 900 1 300 10`, `appendonly no`). This survives pod restarts without mass session invalidation. Full AOF durability is not required since session loss is recoverable.

---

## Repository Structure

A single monorepo (`gpo/gpo-appflowy`) contains everything: upstream source via `git subtree`, GitHub Actions workflows, and Kubernetes manifests. Argo CD is pointed at the `kubernetes/` directory within this repo, which is valid -- an Argo CD `Application` resource simply takes a `repoURL` and `path`.

```
gpo-appflowy/
├── upstream/
│   ├── appflowy-cloud/          # git subtree from AppFlowy-IO/AppFlowy-Cloud
│   └── appflowy-flutter/        # git subtree from AppFlowy-IO/AppFlowy
├── kubernetes/
│   ├── base/                    # Kustomize base manifests (env-agnostic)
│   │   ├── appflowy-cloud/
│   │   ├── appflowy-worker/
│   │   ├── appflowy-search/
│   │   ├── admin-frontend/
│   │   ├── appflowy-web/
│   │   ├── gotrue/
│   │   ├── redis/
│   │   ├── cnpg/
│   │   └── gateway/
│   ├── overlays/
│   │   ├── prod/                # Kustomize overlay -- prod image tags, replicas, secrets refs
│   │   └── stage/               # Kustomize overlay -- stage image tags
│   └── argocd/
│       ├── prod-application.yaml
│       └── stage-application.yaml
└── .github/
    └── workflows/
        ├── sync-upstream.yml    # Weekly: pull upstream changes into subtrees
        ├── build-images.yml     # Weekly: build and push images to Artifact Registry
        └── deploy.yml           # Weekly: update image tags in overlays, commit, trigger Argo CD sync
```

The Argo CD `Application` resources in `kubernetes/argocd/` are added to `gpo-platform-configs` under `kubernetes/argocd-apps/{prod,stage}/application.yaml` alongside the existing apps.

---

## Infrastructure Prerequisites

Before running any manifests, the following must exist. These are one-time setup tasks.

### 1. GCS Buckets

Create two buckets in GCP project `gpo-eng-prod` and `gpo-eng-stage`:

```
gpo-appflowy-prod   (project: gpo-eng-prod,  region: northamerica-northeast2)
gpo-appflowy-stage  (project: gpo-eng-stage, region: northamerica-northeast2)
```

### 2. GCP Service Accounts

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

### 3. pgvector Extension

AppFlowy requires the `vector` extension in Postgres. Enable it after the CNPG cluster is running:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

This can be declared in the CNPG cluster manifest using the `postgresql.parameters` and a bootstrap SQL hook, or applied manually once on first deploy.

### 4. Google Workspace OAuth Client

In Google Cloud Console for `gpo-eng-prod`:

1. APIs & Services > Credentials > Create OAuth 2.0 Client ID
2. Application type: Web application
3. Authorised redirect URIs:
   - `https://appflowy.gpotools.ca/gotrue/callback`
   - `https://appflowy.gpotoolsstaging.ca/gotrue/callback`
4. Note the client ID and secret -- these go into the Kubernetes Secret for GoTrue.

### 5. Cloudflare DNS Records

Add DNS records in the `gpotools.ca` and `gpotoolsstaging.ca` zones pointing to the GKE Gateway external IP. Follow the existing pattern in `kubernetes/gateway/{prod,stage}/`.

### 6. Artifact Registry Repository

Verify the registry `northamerica-northeast2-docker.pkg.dev/<project>/gpo` exists and that the GitHub Actions service account has `roles/artifactregistry.writer`.

---

## Kubernetes Manifests

All manifests use Kustomize. Base manifests are environment-agnostic; overlays patch image tags and environment-specific values.

### CNPG Cluster (`kubernetes/base/cnpg/cluster.yaml`)

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: appflowy-pg
  namespace: appflowy
spec:
  instances: 1
  storage:
    size: 8Gi
  postgresql:
    parameters:
      shared_preload_libraries: "vector"
  bootstrap:
    initdb:
      database: appflowy
      owner: appflowy
      postInitSQL:
        - CREATE EXTENSION IF NOT EXISTS vector;
```

### Redis (`kubernetes/base/redis/`)

**Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: appflowy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          args:
            - redis-server
            - --save
            - "900 1"
            - --save
            - "300 10"
            - --appendonly
            - "no"
            - --requirepass
            - $(REDIS_PASSWORD)
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: appflowy-secrets
                  key: redis-password
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
      volumes:
        - name: redis-data
          persistentVolumeClaim:
            claimName: redis-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: appflowy
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: appflowy
spec:
  selector:
    app: redis
  ports:
    - port: 6379
```

### GoTrue (`kubernetes/base/gotrue/`)

GoTrue runs as a Deployment. Key environment variables -- set via Secret:

```yaml
# In appflowy-secrets Secret:
GOTRUE_DB_DRIVER: postgres
DATABASE_URL: "postgres://appflowy:<password>@appflowy-pg-rw.appflowy.svc:5432/appflowy"
GOTRUE_SITE_URL: "https://appflowy.gpotools.ca"
GOTRUE_URI_ALLOW_LIST: "https://appflowy.gpotools.ca/**"
GOTRUE_JWT_SECRET: "<random-256-bit-secret>"
GOTRUE_JWT_EXP: "3600"

# Disable email signup -- Google SSO only
GOTRUE_MAILER_AUTOCONFIRM: "false"
GOTRUE_DISABLE_SIGNUP: "true"       # prevents email/password registration
GOTRUE_EXTERNAL_EMAIL_ENABLED: "false"

# Google OAuth
GOTRUE_EXTERNAL_GOOGLE_ENABLED: "true"
GOTRUE_EXTERNAL_GOOGLE_CLIENT_ID: "<oauth-client-id>"
GOTRUE_EXTERNAL_GOOGLE_SECRET: "<oauth-client-secret>"
GOTRUE_EXTERNAL_GOOGLE_REDIRECT_URI: "https://appflowy.gpotools.ca/gotrue/callback"

# Restrict to gpo.ca domain -- GoTrue supports this via allowed email domains
GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS: "gpo.ca"
```

> **Note:** `GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS` restricts Google SSO to accounts from `gpo.ca` only. Verify this env var is supported in AppFlowy's GoTrue fork; if not, domain restriction can be enforced at the AppFlowy application layer or via a GoTrue hook.

### AppFlowy Cloud (`kubernetes/base/appflowy-cloud/`)

Key env vars (remaining vars from `deploy.env` carry over):

```yaml
APPFLOWY_ENVIRONMENT: production
APPFLOWY_DATABASE_URL: "postgres://appflowy:<password>@appflowy-pg-rw.appflowy.svc:5432/appflowy"
APPFLOWY_REDIS_URI: "redis://:$(REDIS_PASSWORD)@redis.appflowy.svc:6379"
APPFLOWY_GOTRUE_BASE_URL: "http://gotrue.appflowy.svc:9999"
APPFLOWY_BASE_URL: "https://appflowy.gpotools.ca"

# GCS storage (replaces MinIO)
APPFLOWY_S3_USE_MINIO: "false"
AWS_ACCESS_KEY: ""                  # not used with Workload Identity
AWS_SECRET: ""
APPFLOWY_S3_BUCKET: "gpo-appflowy-prod"
APPFLOWY_S3_REGION: "northamerica-northeast2"

# AI (optional -- omit to disable)
AI_OPENAI_API_KEY: "<openai-key>"
```

AppFlowy's S3 client uses the AWS SDK. To use GCS with Workload Identity, set `APPFLOWY_S3_USE_MINIO: "false"` and configure the GKE service account annotation. The AWS SDK respects Application Default Credentials when no key is provided and GCS exposes an S3-compatible endpoint at `storage.googleapis.com`.

> **Verify:** Confirm AppFlowy's S3 client supports GCS path-style or virtual-hosted URLs. If not, deploy a lightweight MinIO gateway in front of GCS as a fallback -- this is a single-container proxy, not a storage system.

### Gateway (`kubernetes/base/gateway/`)

Follow the existing pattern in `gpo-platform-configs/kubernetes/gateway/`. Create an `HTTPRoute` routing all traffic for `appflowy.gpotools.ca` to the `appflowy-web` service, with path-based routing for `/api`, `/ws`, `/gotrue`, and `/minio` sub-paths forwarding to their respective services.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: appflowy
  namespace: appflowy
spec:
  parentRefs:
    - name: <existing-gateway-name>      # match the gateway name in gpo-platform-configs
      namespace: <gateway-namespace>
  hostnames:
    - appflowy.gpotools.ca
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: appflowy-cloud
          port: 8000
    - matches:
        - path:
            type: PathPrefix
            value: /ws
      backendRefs:
        - name: appflowy-cloud
          port: 8000
    - matches:
        - path:
            type: PathPrefix
            value: /gotrue
      backendRefs:
        - name: gotrue
          port: 9999
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: appflowy-web
          port: 80
```

### Kustomize Overlays

`kubernetes/overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: appflowy
resources:
  - ../../base
images:
  - name: appflowy-cloud
    newName: northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/appflowy-cloud
    newTag: <SHA>   # replaced by deploy workflow
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
  - path: gotrue-patch.yaml          # prod-specific GoTrue env (site URL, redirect URI)
  - path: appflowy-cloud-patch.yaml  # prod-specific env (base URL, bucket name)
```

---

## Argo CD Applications

Add these two `Application` resources to `gpo-platform-configs/kubernetes/argocd-apps/prod/application.yaml` and the stage equivalent, following the existing pattern:

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
    path: kubernetes/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: appflowy
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

`automated.selfHeal: true` means Argo CD will re-apply on any drift. The weekly deploy workflow commits updated image tags to `main`, and Argo CD picks them up automatically -- no manual promotion step.

---

## GitHub Actions Workflows

### Weekly Schedule

All three workflows run in sequence on Sunday at 07:00 UTC (03:00 ET):

```
sync-upstream (07:00 UTC)
  → on success: build-images (07:05 UTC, triggered by workflow_run)
    → on success: deploy (triggered by workflow_run)
```

### 1. Sync Upstream (`sync-upstream.yml`)

```yaml
name: Sync upstream AppFlowy sources

on:
  schedule:
    - cron: '0 7 * * 0'   # Sunday 07:00 UTC
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Sync AppFlowy-Cloud subtree
        run: |
          git subtree pull \
            --prefix upstream/appflowy-cloud \
            https://github.com/AppFlowy-IO/AppFlowy-Cloud.git main \
            --squash \
            -m "chore: sync AppFlowy-Cloud $(date -u +%Y-%m-%d)"

      - name: Sync AppFlowy Flutter subtree
        run: |
          git subtree pull \
            --prefix upstream/appflowy-flutter \
            https://github.com/AppFlowy-IO/AppFlowy.git main \
            --squash \
            -m "chore: sync AppFlowy Flutter $(date -u +%Y-%m-%d)"

      - name: Push changes
        run: git push origin main
```

### 2. Build Images (`build-images.yml`)

```yaml
name: Build and push AppFlowy images

on:
  workflow_run:
    workflows: ["Sync upstream AppFlowy sources"]
    types: [completed]
  workflow_dispatch:

jobs:
  build:
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # for Workload Identity Federation to Artifact Registry

    strategy:
      matrix:
        include:
          - name: appflowy-cloud
            context: upstream/appflowy-cloud
            dockerfile: upstream/appflowy-cloud/Dockerfile
          - name: appflowy-worker
            context: upstream/appflowy-cloud
            dockerfile: upstream/appflowy-cloud/services/appflowy-worker/Dockerfile
          - name: appflowy-search
            context: upstream/appflowy-cloud
            dockerfile: upstream/appflowy-cloud/services/appflowy-search/Dockerfile
          - name: admin-frontend
            context: upstream/appflowy-cloud/admin_frontend
            dockerfile: upstream/appflowy-cloud/admin_frontend/Dockerfile
          - name: appflowy-web
            context: upstream/appflowy-cloud
            dockerfile: upstream/appflowy-cloud/services/appflowy-web/Dockerfile

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_ARTIFACT_REGISTRY_SA }}

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker northamerica-northeast2-docker.pkg.dev

      - name: Build and push ${{ matrix.name }}
        uses: docker/build-push-action@v5
        with:
          context: ${{ matrix.context }}
          file: ${{ matrix.dockerfile }}
          push: true
          tags: |
            northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/${{ matrix.name }}:${{ github.sha }}
            northamerica-northeast2-docker.pkg.dev/gpo-eng-prod/gpo/${{ matrix.name }}:latest
          build-args: PROFILE=release
          cache-from: type=gha,scope=${{ matrix.name }}
          cache-to: type=gha,mode=max,scope=${{ matrix.name }}
```

> **Note on build time:** The `appflowy-cloud` Rust build is the bottleneck at 20-40 min on a cold cache. GitHub-hosted runners get 10 GB of Actions cache. With `cargo-chef` caching in the Dockerfile and `cache-from: type=gha`, subsequent weekly builds should drop to 5-10 min per image. All five images run in parallel via the matrix.

### 3. Deploy (`deploy.yml`)

```yaml
name: Deploy AppFlowy

on:
  workflow_run:
    workflows: ["Build and push AppFlowy images"]
    types: [completed]
  workflow_dispatch:
    inputs:
      sha:
        description: 'Image SHA to deploy (defaults to latest build)'
        required: false

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Determine image SHA
        id: sha
        run: |
          SHA="${{ inputs.sha || github.event.workflow_run.head_sha || github.sha }}"
          echo "sha=${SHA}" >> $GITHUB_OUTPUT

      - name: Update image tags in prod overlay
        run: |
          SHA=${{ steps.sha.outputs.sha }}
          for IMAGE in appflowy-cloud appflowy-worker appflowy-search admin-frontend appflowy-web; do
            sed -i "s|newTag:.*# ${IMAGE}|newTag: ${SHA} # ${IMAGE}|g" \
              kubernetes/overlays/prod/kustomization.yaml
            sed -i "s|newTag:.*# ${IMAGE}|newTag: ${SHA} # ${IMAGE}|g" \
              kubernetes/overlays/stage/kustomization.yaml
          done

      - name: Commit and push
        run: |
          git config user.name "gpo-bot"
          git config user.email "bot@gpo.ca"
          git add kubernetes/overlays/
          git commit -m "chore: deploy appflowy $(date -u +%Y-%m-%d) sha=${SHA:0:7}" \
            --allow-empty
          git push origin main
```

Argo CD's `automated` sync policy picks up the commit and rolls out both environments. No further action needed.

---

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

---

## First-Time Deployment Sequence

Run these steps once before Argo CD takes over ongoing deploys.

1. **Create GCS buckets** (prod and stage) per Infrastructure Prerequisites above.
2. **Create GCP service accounts** and configure Workload Identity bindings.
3. **Create Google OAuth client** in GCP Console. Note client ID and secret.
4. **Add DNS records** in Cloudflare for `appflowy.gpotools.ca` and `appflowy.gpotoolsstaging.ca`.
5. **Create the `gpo-appflowy` repo** under the `gpo` org.
6. **Add upstream subtrees:**
   ```bash
   git subtree add --prefix upstream/appflowy-cloud \
     https://github.com/AppFlowy-IO/AppFlowy-Cloud.git main --squash
   git subtree add --prefix upstream/appflowy-flutter \
     https://github.com/AppFlowy-IO/AppFlowy.git main --squash
   ```
7. **Add all manifests and workflows** from this document.
8. **Run `build-images` workflow manually** (`workflow_dispatch`) to produce the first set of images.
9. **Create Kubernetes secrets** in both clusters:
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
   # Repeat for gpo-stage context
   ```
10. **Add Argo CD Application resources** to `gpo-platform-configs` for both environments.
11. **Trigger Argo CD sync** and verify all pods reach `Running`.
12. **Verify GoTrue** by navigating to `https://appflowy.gpotools.ca/gotrue/health`.
13. **Verify AppFlowy web** by navigating to `https://appflowy.gpotools.ca` and completing Google SSO with a `gpo.ca` account.
14. **Run the weekly workflow manually** to confirm the full sync → build → deploy pipeline works end to end.

---

## Risks

| Risk | Severity |
|---|---|
| AppFlowy relicenses or hollows out the AGPL repo | **High** -- monitor commits; evaluate Outline + Plane as fallback |
| Upstream breaks compatibility between server and clients | **Medium** -- pin client and server to matching commits; fork both together |
| Flutter client source falls behind released binaries | **Medium** -- already occurring; fork before gap widens further |
| GCS S3 compatibility with AppFlowy's AWS SDK client | **Medium** -- verify on first deploy; MinIO gateway is the fallback |
| `GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS` support in AppFlowy's GoTrue fork | **Medium** -- verify; fallback is application-layer domain check |
| Redis pod restart invalidates sessions | **Low** -- mitigated by RDB persistence; users re-auth via Google SSO |
| GitHub Actions cache (10 GB) eviction causing slow Rust builds | **Low** -- per-image cache scopes prevent single-image eviction from affecting others |
| Ongoing maintenance tracking a fast-moving upstream | **Medium** -- budget ~2-4h/month |

---

## Effort Estimate

| Task | Estimate |
|---|---|
| Repo setup, subtrees, initial build | 1 day |
| Kubernetes manifests (base + overlays) | 2-3 days |
| GitHub Actions workflows | 1 day |
| Argo CD integration into platform configs | 0.5 days |
| GCP prereqs (buckets, SA, OAuth, DNS) | 0.5 days |
| First-time deploy and smoke testing | 1 day |
| Apple Developer account (iOS contingency) | $99 USD/year |
| Ongoing maintenance | ~2-4h/month |

