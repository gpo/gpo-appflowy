# Overview: AppFlowy AGPL Deployment on GKE

Part of the [AppFlowy AGPL build design](./appflowy-agpl-build-design.md). See the [documentation index](../README.md) and the [roadmap](../roadmap.md) for the execution plan.

**Status:** Draft
**Upstream:** https://github.com/AppFlowy-IO/AppFlowy-Cloud (AGPL-3.0)
**Monorepo:** `github.com/gpo/gpo-appflowy`
**Platform config:** `github.com/gpo/gpo-platform-configs` (existing)

## The Core Finding

The user limit is not in the AGPL source. Direct inspection of `src/biz/user/user_verify.rs`, `src/biz/workspace/ops.rs`, and a full `src/` grep for `max_users`, `license`, `plan_limit`, and related terms found nothing. The `"Free plan limits - max_users: 1"` message in community bug reports comes entirely from the closed Docker Hub binary, not this codebase.

Building from source and substituting our own images for the four closed ones is the complete solution.

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
| `minio/minio`, `pgvector/pgvector`, `redis` | Fully open | Not used, replaced by GCS, CNPG, in-cluster Redis |

## What We Lose Without the Commercial Binary

We lose:

- AI response quotas managed by Stripe. The AGPL source has no AI billing enforcement; supply an OpenAI API key directly via env vars and it works without limits.
- Stripe billing and subscription management UI, which is irrelevant for internal use.
- Priority support, the Team/Enterprise SLA.

We keep everything GPO actually needs:

- Unlimited users and workspaces
- Real-time collaborative editing
- Databases (grid, kanban, calendar)
- Wiki and documents
- Semantic search via `appflowy_search` (uses pgvector, not Elasticsearch)
- Google Workspace SSO
- GCS file storage
- Full mobile and desktop client support
