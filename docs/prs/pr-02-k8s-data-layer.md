# PR 02: Kubernetes Base, Data Layer

**Depends on:** PR 01
**Blocks:** PR 04 (services need Postgres and Redis), PR 06 (overlays)
**Estimate:** ~0.5 day

## Goal

Stand up the stateful backing services: the `appflowy` namespace, a CloudNativePG (CNPG) Postgres cluster with the `vector` extension, and a Redis instance with RDB persistence.

## Scope

```
kubernetes/base/cnpg/cluster.yaml
kubernetes/base/cnpg/kustomization.yaml
kubernetes/base/redis/deployment.yaml      # Deployment + PVC + Service
kubernetes/base/redis/kustomization.yaml
kubernetes/base/namespace.yaml             # optional; CreateNamespace=true also handles this
```

## CNPG Cluster

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

## Redis

Deployment with RDB persistence, password from the `appflowy-secrets` Secret, a 1Gi PVC, and a ClusterIP Service on 6379. Full manifest is in the [original design doc](../design/appflowy-agpl-build-design.md#redis-kubernetesbaseredis). Key args:

```
redis-server --save "900 1" --save "300 10" --appendonly no --requirepass $(REDIS_PASSWORD)
```

`REDIS_PASSWORD` is injected from `secretKeyRef: { name: appflowy-secrets, key: redis-password }`.

## Acceptance Criteria

- `kustomize build kubernetes/base/cnpg` and `.../redis` render without error.
- Manifests pass `kubeconform` (CNPG CRDs may need a schema location flag).
- Redis args produce RDB-only persistence (`appendonly no`), per [02-architecture.md](../design/02-architecture.md#redis-usage).
- The CNPG cluster declares the `vector` extension via `postInitSQL`.

## Dependencies and Assumptions

- The CNPG operator is already installed in the target clusters (it is part of `gpo-platform-configs`; confirm before merge).
- The `appflowy-secrets` Secret is created out of band ([PR 09](./pr-09-prereqs-and-secrets.md)). This PR only references it.
