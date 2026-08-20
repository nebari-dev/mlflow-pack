# Nebari MLflow Pack

Deploys [MLflow](https://mlflow.org/) on [Nebari](https://nebari.dev) with
Keycloak authentication, PostgreSQL backend storage, and automatic TLS.

## Quick Start

1. **Create the PostgreSQL credentials secret:**

   ```bash
   kubectl create namespace mlflow

   kubectl create secret generic mlflow-pack-postgresql \
     --namespace mlflow \
     --from-literal=password="$(openssl rand -base64 32)" \
     --from-literal=postgres-password="$(openssl rand -base64 32)"
   ```

   The secret name **must** be `<release-name>-postgresql` and contain keys
   `password` (mlflow DB user) and `postgres-password` (superuser).

2. **Copy the example ArgoCD Application and edit it for your cluster:**

   ```bash
   cp examples/nebari-values.yaml /path/to/your/gitops-repo/apps/mlflow-pack.yaml
   ```

   Update `nebariapp.hostname`, `nebariapp.keycloakHostname`, and
   `mlflow.postgresql.primary.persistence.storageClass` for your environment.

3. **Add `mlflow.<your-domain>` to your gateway certificate and DNS.**

See [examples/nebari-values.yaml](examples/nebari-values.yaml) for
the full ArgoCD Application manifest.

## Connecting JupyterHub

To allow [nebari-data-science-pack](https://github.com/nebari-dev/nebari-data-science-pack)
notebooks to log experiments to MLflow, add the following to your
data-science-pack ArgoCD Application values:

```yaml
jupyterhub:
  singleuser:
    extraEnv:
      MLFLOW_TRACKING_URI: "http://mlflow-pack.mlflow.svc.cluster.local:80"
    networkPolicy:
      egress:
        - ports:
            - port: 5000
              protocol: TCP
          to:
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: mlflow
```

The egress rule uses port **5000** (the pod port) because NetworkPolicy operates
at the pod IP level, not the ClusterIP service level (which maps 80 to 5000).

After applying, existing JupyterLab sessions must be restarted (stop/start from
the hub control panel) to pick up the new environment variable and NetworkPolicy.

### Verify from a notebook

```python
import mlflow
mlflow.set_experiment("test")
with mlflow.start_run():
    mlflow.log_param("framework", "pytorch")
    mlflow.log_metric("accuracy", 0.95)
print("Run ID:", mlflow.last_active_run().info.run_id)
```

## Configuration

### PostgreSQL

By default, this chart bundles a Bitnami PostgreSQL instance. For dev/testing
you can pass the password inline instead of creating a secret:

```bash
helm install mlflow-pack . \
  --set mlflow.postgresql.auth.password=my-dev-password
```

**Do not use inline passwords in production or commit them to a gitops repository.**

To disable PostgreSQL and use in-memory SQLite (data lost on pod restart):

```yaml
mlflow:
  postgresql:
    enabled: false
```

### Allowed Hosts

The chart automatically whitelists the NebariApp hostname and the
cluster-internal service name via `MLFLOW_SERVER_ALLOWED_HOSTS`. To allow
additional hosts:

```yaml
security:
  additionalAllowedHosts:
    - custom-alias.internal
```

## Troubleshooting

### NebariApp not ready

```bash
kubectl get nebariapp -n mlflow
kubectl describe nebariapp -n mlflow
```

Check conditions: `RoutingReady`, `TLSReady`, `AuthReady` should all be `True`.

## Documentation

The docs site lives in [`docs/`](docs/) and is built with [Astro](https://astro.build) +
[Starlight](https://starlight.astro.build) using the shared `@nebari/starlight` theme. It
deploys to [packs.nebari.dev/mlflow-pack/](https://packs.nebari.dev/mlflow-pack/) on every
merge to `main`; pull requests that touch `docs/` get a preview URL posted as a comment.

```bash
cd docs
npm ci
npm run dev     # dev server with hot reload at http://localhost:4321
npm run build   # static build into docs/dist/
npm test        # unit tests
```

Pages live in `docs/src/content/docs/` - each `.md` or `.mdx` file becomes a page, and the
sidebar is configured in `docs/astro.config.mjs`. See [`docs/README.md`](docs/README.md) for
details.

## License

Apache 2.0 - see [LICENSE](LICENSE).
