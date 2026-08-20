---
title: Getting started
description: Create the PostgreSQL secret, install the chart, and confirm MLflow is up.
---

## Prerequisites

- A Kubernetes cluster and Helm 3.
- For the full Nebari path: the
  [nebari-operator](https://github.com/nebari-dev/nebari-operator) (it provides the
  `NebariApp` CRD), Envoy Gateway, cert-manager with a cluster issuer, and a Keycloak realm.
- A StorageClass for the PostgreSQL PVC.

To try MLflow without any of the Nebari pieces, skip to
[Standalone deployment](/standalone/).

## 1. Create the PostgreSQL credentials secret

The bundled PostgreSQL reads its passwords from a Secret you create in advance.

```bash
kubectl create namespace mlflow

kubectl create secret generic mlflow-pack-postgresql \
  --namespace mlflow \
  --from-literal=password="$(openssl rand -base64 32)" \
  --from-literal=postgres-password="$(openssl rand -base64 32)"
```

:::caution[The name and keys are a contract, not a convention]
The Secret must be named **`<release-name>-postgresql`** — `mlflow-pack-postgresql` for a
release called `mlflow-pack` — and must contain exactly the keys `password` (the `mlflow`
database user) and `postgres-password` (the superuser). Get either wrong and the
PostgreSQL pod crash-loops on a key it cannot find.
:::

Then reference it:

```yaml
mlflow:
  postgresql:
    auth:
      existingSecret: mlflow-pack-postgresql
```

For a throwaway cluster you can skip the secret and pass the password inline —
`--set mlflow.postgresql.auth.password=dev-only` — but never in production or a GitOps
repository. More detail in [PostgreSQL backend](/postgresql/).

## 2. Install

```bash
helm install mlflow-pack . \
  --namespace mlflow \
  --set nebariapp.hostname=mlflow.example.com \
  --set nebariapp.keycloakHostname=keycloak.example.com \
  --set mlflow.postgresql.auth.existingSecret=mlflow-pack-postgresql
```

On a GitOps cluster, use the Argo CD `Application` instead — see
[Deploying on Nebari](/deployment/).

:::caution[`hostname` is required]
With `nebariapp.enabled: true` (the default) and no `nebariapp.hostname`, the chart refuses
to render: `nebariapp.hostname is required when nebariapp.enabled is true`.
:::

## 3. Point DNS and the certificate at the hostname

Add `mlflow.<your-domain>` to your gateway certificate and to DNS. Until both exist the
`NebariApp` reports `RoutingReady` but the browser cannot reach it.

## What gets deployed

| Workload | Kind | Purpose |
|---|---|---|
| `mlflow-pack` | Deployment | MLflow server, container port `5000`, service port `80` |
| `mlflow-pack-postgresql` | StatefulSet | Backend store, 8Gi PVC |
| `nebari-mlflow-allowed-hosts` | Secret | `MLFLOW_SERVER_ALLOWED_HOSTS`, injected via `envFrom` |
| `mlflow-pack` | NebariApp | Routing, TLS, and Keycloak client |

Note the service is named after the **release**, not `<release>-mlflow` — the community
chart's fullname helper collapses when the release name contains the chart name. The
`nebariapp.service.name` default follows the same helper, so the two always agree.

## Verify

```bash
kubectl -n mlflow get pods
kubectl -n mlflow rollout status deployment/mlflow-pack

# Health endpoint, straight at the pod
kubectl -n mlflow port-forward svc/mlflow-pack 5080:80 &
curl -sf http://localhost:5080/health && echo OK
kill %1
```

Then the routing layer:

```bash
kubectl -n mlflow get nebariapp
kubectl -n mlflow describe nebariapp mlflow-pack
```

`RoutingReady`, `TLSReady`, and `AuthReady` should all be `True`. If they are not,
[Troubleshooting](/troubleshooting/) maps each one to its usual cause.

Open `https://mlflow.example.com`. Keycloak takes the login, then MLflow's UI loads.

## Next

- Wire up notebooks: [Connecting JupyterHub](/jupyterhub/)
- Make artifacts durable — the default is not:
  [Artifact storage](/artifact-storage/)
