# Helm quick commands (Geargo chart)

This short reference shows the most common Helm commands you'll use with the `geargo` chart.

## Validate and render templates (no cluster changes)

```bash
# Lint the chart
helm lint .

# Render templates locally to inspect what will be applied
helm template geargo . --namespace geargo > rendered.yaml
less rendered.yaml
```

## Install or upgrade the chart

Replace `<ECR_REPO>` and `<IMAGE_TAG>` with your values. Do NOT put secrets in version-controlled files — use CI secrets or `--set` at deploy time.

```bash
helm upgrade --install geargo . -n geargo --create-namespace \
  --set image.repository=<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<ECR_REPO> \
  --set image.tag=<IMAGE_TAG> \
  --set postgres.enabled=true \
  --set postgres.password='${POSTGRES_PASSWORD}'
```

If you prefer not to pass the password on the command line, create a Kubernetes Secret and point `database.secretName` at it:

```bash
kubectl -n geargo create secret generic geargo-db-secret \
  --from-literal=username=geargo --from-literal=password='StrongPasswordHere' \
  --from-literal=database=geargo_db \
  --from-literal=database-url="postgres://geargo:StrongPasswordHere@geargo-postgres:5432/geargo_db"

helm upgrade --install geargo . -n geargo --create-namespace \
  --set database.secretName=geargo-db-secret \
  --set postgres.enabled=false \
  --set image.repository=<...> --set image.tag=<...>
```

## Useful inspection & debug commands

```bash
# Show releases
helm list -n geargo

# Show release values
helm get values geargo -n geargo

# Show generated manifests for a release
helm get manifest geargo -n geargo | less

# Diff (requires helm-diff plugin): see what will change
helm plugin install https://github.com/databus23/helm-diff || true
helm diff upgrade geargo . -n geargo --allow-unreleased
```

## Rollback and uninstall

```bash
# Rollback to a previous revision (e.g. 1)
helm rollback geargo 1 -n geargo

# Uninstall
helm uninstall geargo -n geargo
kubectl -n geargo delete namespace geargo --ignore-not-found
```

## CI notes (GitHub Actions)

- Store sensitive values (AWS creds, `POSTGRES_PASSWORD`, `KUBE_CONFIG_DATA`) as repository Secrets.
- The included workflow sets `image.repository` and `image.tag` from the pushed image and passes `postgres.password` from `POSTGRES_PASSWORD`.
- For more secure secret handling in the cluster, consider SealedSecrets or External Secrets rather than embedding in Helm values.

If you want, I can add a small `helm/` helper script (`deploy.sh`) to the repo that wraps these commands and validates required env variables.
