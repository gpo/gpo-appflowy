# PR 03: Kubernetes Base, GoTrue Auth

**Depends on:** PR 02 (needs Postgres)
**Blocks:** PR 04 (appflowy-cloud calls GoTrue), PR 05 (gateway routes `/gotrue`)
**Estimate:** ~0.5 day

## Goal

Deploy GoTrue (Supabase Auth fork, open image) configured for Google Workspace SSO only, with email signup disabled and sign-in restricted to the `gpo.ca` domain.

## Scope

```
kubernetes/base/gotrue/deployment.yaml
kubernetes/base/gotrue/service.yaml          # ClusterIP on 9999
kubernetes/base/gotrue/configmap.yaml        # non-secret env
kubernetes/base/gotrue/kustomization.yaml
```

Use the upstream `appflowyinc/gotrue` image (open). Do not build this one.

## Configuration

Non-secret env via ConfigMap, secret env via `appflowy-secrets`. Environment-specific values (site URL, redirect URI) are patched in the overlays ([PR 06](./pr-06-kustomize-overlays.md)); base carries env-agnostic defaults.

```
GOTRUE_DB_DRIVER: postgres
DATABASE_URL: postgres://appflowy:<password>@appflowy-pg-rw.appflowy.svc:5432/appflowy
GOTRUE_JWT_EXP: "3600"
GOTRUE_MAILER_AUTOCONFIRM: "false"
GOTRUE_DISABLE_SIGNUP: "true"
GOTRUE_EXTERNAL_EMAIL_ENABLED: "false"
GOTRUE_EXTERNAL_GOOGLE_ENABLED: "true"
GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS: "gpo.ca"
```

Secret-sourced: `GOTRUE_JWT_SECRET`, `GOTRUE_EXTERNAL_GOOGLE_CLIENT_ID`, `GOTRUE_EXTERNAL_GOOGLE_SECRET`, and the password inside `DATABASE_URL`.

Overlay-patched per environment: `GOTRUE_SITE_URL`, `GOTRUE_URI_ALLOW_LIST`, `GOTRUE_EXTERNAL_GOOGLE_REDIRECT_URI`.

## Verification Task (carry into first deploy)

Confirm `GOTRUE_EXTERNAL_GOOGLE_ALLOWED_DOMAINS` is honoured by AppFlowy's GoTrue fork. If it is not, fall back to an application-layer domain check or a GoTrue hook. This is a tracked [Medium risk](../design/06-secrets-deployment-risks.md#risks).

## Acceptance Criteria

- `kustomize build kubernetes/base/gotrue` renders cleanly and passes `kubeconform`.
- No secret values are committed; all sensitive env uses `secretKeyRef`.
- Signup paths are disabled; only Google SSO is enabled.
