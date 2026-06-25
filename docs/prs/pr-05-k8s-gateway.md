# PR 05: Kubernetes Base, Gateway and HTTPRoute

**Depends on:** PR 03 (gotrue), PR 04 (appflowy-cloud, appflowy-web)
**Blocks:** PR 06 (overlays reference the route host)
**Estimate:** ~0.5 day

## Goal

Expose AppFlowy through the existing GKE Gateway with an `HTTPRoute` that path-routes API, websocket, auth, and web traffic.

## Scope

```
kubernetes/base/gateway/httproute.yaml
kubernetes/base/gateway/kustomization.yaml
```

## HTTPRoute

Routes for `appflowy.gpotools.ca` (hostname patched per env in the overlay):

| Path prefix | Backend | Port |
|---|---|---|
| `/api` | `appflowy-cloud` | 8000 |
| `/ws` | `appflowy-cloud` | 8000 |
| `/gotrue` | `gotrue` | 9999 |
| `/` | `appflowy-web` | 80 |

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: appflowy
  namespace: appflowy
spec:
  parentRefs:
    - name: <existing-gateway-name>      # match gpo-platform-configs
      namespace: <gateway-namespace>
  hostnames:
    - appflowy.gpotools.ca
  rules:
    - matches: [{ path: { type: PathPrefix, value: /api } }]
      backendRefs: [{ name: appflowy-cloud, port: 8000 }]
    - matches: [{ path: { type: PathPrefix, value: /ws } }]
      backendRefs: [{ name: appflowy-cloud, port: 8000 }]
    - matches: [{ path: { type: PathPrefix, value: /gotrue } }]
      backendRefs: [{ name: gotrue, port: 9999 }]
    - matches: [{ path: { type: PathPrefix, value: / } }]
      backendRefs: [{ name: appflowy-web, port: 80 }]
```

## Acceptance Criteria

- `parentRefs` name and namespace match the live gateway in `gpo-platform-configs` (confirm before merge; left as placeholders here).
- Most specific path prefixes are ordered before `/` so they win.
- Renders via `kustomize build` and passes `kubeconform` with Gateway API schemas.

## Notes

The hostname (`appflowy.gpotools.ca` vs. `appflowy.gpotoolsstaging.ca`) is environment-specific and is patched in the overlays ([PR 06](./pr-06-kustomize-overlays.md)).
