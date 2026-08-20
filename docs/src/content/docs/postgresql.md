---
title: PostgreSQL backend
description: The credentials secret contract, sizing, migrations, and the SQLite fallback.
---

MLflow's backend store holds every run, parameter, metric, tag, and the model registry.
This chart enables the bundled PostgreSQL by default because the alternative loses all of
it on restart.

```yaml
mlflow:
  postgresql:
    enabled: true
    image:
      tag: "17.5.0"
    auth:
      username: mlflow
      database: mlflow
      existingSecret: ""
    primary:
      persistence:
        enabled: true
        size: 8Gi
```

When `postgresql.enabled` is true the upstream chart wires `PGHOST`, `PGPORT`, `PGDATABASE`,
`PGUSER`, and `PGPASSWORD` into the MLflow container and sets
`--backend-store-uri=postgresql://` on the server. Nothing to configure by hand.

## The credentials secret

```bash
kubectl create secret generic mlflow-pack-postgresql \
  --namespace mlflow \
  --from-literal=password="$(openssl rand -base64 32)" \
  --from-literal=postgres-password="$(openssl rand -base64 32)"
```

```yaml
mlflow:
  postgresql:
    auth:
      existingSecret: mlflow-pack-postgresql
```

:::caution[The name and keys are exact]
- **Name** — `<release-name>-postgresql`. A release called `mlflow-pack` needs
  `mlflow-pack-postgresql`.
- **Keys** — `password` (the `mlflow` user) and `postgres-password` (the superuser). Both
  are required.

A missing key makes the PostgreSQL pod crash-loop referencing a key name, which is at least
a legible error. A misnamed *secret* is worse: the chart generates a random password
instead, and the failure only appears later as an authentication error from MLflow.
:::

### Inline passwords

For a throwaway cluster:

```bash
helm install mlflow-pack . --set mlflow.postgresql.auth.password=dev-only
```

Never in production, and never in a values file committed to a GitOps repository — Helm
values are readable by anyone with access to the release.

### Rotating the password

Update the Secret, then restart both workloads:

```bash
kubectl -n mlflow rollout restart statefulset/mlflow-pack-postgresql
kubectl -n mlflow rollout restart deployment/mlflow-pack
```

Changing the Secret alone changes nothing — the values are read into the environment at
pod start.

## Storage

The default 8Gi PVC is generous for metadata: runs, params, and metrics are small rows.
Growth comes from *run volume*, not model size — models live in the
[artifact store](/artifact-storage/).

Set the storage class explicitly on any cluster where the default is not what you want.
PVCs cannot be reassigned to a different class after creation:

```yaml
mlflow:
  postgresql:
    primary:
      persistence:
        size: 20Gi
        storageClass: fast-ssd
```

## Migrations

```yaml
mlflow:
  backendStore:
    databaseMigration: true
    databaseConnectionCheck: true
```

`databaseMigration` runs MLflow's schema migration at startup, so a chart upgrade that
bumps MLflow also updates the database. `databaseConnectionCheck` adds an init container
that waits for PostgreSQL to accept connections, avoiding a crash-loop while the database
is still starting.

:::caution[Migrations are one-way]
Rolling the chart back to an older MLflow does not roll the schema back, and an older
server may not start against a newer schema. Snapshot the PostgreSQL volume before a major
version bump.
:::

## Backups

The chart does not back up the database. On a Nebari cluster with
[longhorn-backup-pack](https://packs.nebari.dev/longhorn-backup-pack/) installed, the
PostgreSQL PVC is covered by the cluster-wide schedule if it is on the default
StorageClass — worth confirming rather than assuming:

```bash
kubectl -n longhorn-system get volumes.longhorn.io \
  -l recurring-job-group.longhorn.io/default=enabled
```

For a logical backup:

```bash
kubectl -n mlflow exec -it statefulset/mlflow-pack-postgresql -- \
  pg_dump -U mlflow mlflow > mlflow-backup.sql
```

A complete restore needs both halves — the database *and* the artifact bucket. Restoring
one without the other produces runs that reference missing models, or orphaned files no run
points at.

## Using an external PostgreSQL

Disable the bundled instance and supply a connection string:

```yaml
mlflow:
  postgresql:
    enabled: false
  backendStore:
    postgres:
      enabled: true
      host: postgres.example.com
      port: 5432
      database: mlflow
      user: mlflow
      # password or existingSecret, per the upstream chart
```

Check the [community chart values](https://github.com/community-charts/helm-charts/tree/main/charts/mlflow)
for the exact shape — that block belongs to the subchart, not this one.

## The SQLite fallback

```yaml
mlflow:
  postgresql:
    enabled: false
```

MLflow then falls back to SQLite on the container filesystem. There is no PVC, so **every
experiment is lost on restart**, along with the artifacts. It is fine for a five-minute
demo or CI — the chart's own integration test uses it — and nothing else.

`examples/standalone-values.yaml` uses this combination deliberately; see
[Standalone deployment](/standalone/).
