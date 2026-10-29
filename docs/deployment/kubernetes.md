# Kubernetes

*Community edition, Helm chart 0.1.0. Installed and tested on k3s 1.31: install,
`helm test`, a CSV query over MCP, a refused file read, and a pod restart that kept the
registry and audit log.*

## What you get

- **One pod, by design.** The source registry (SQLite) and the audit log are files on one
  ReadWriteOnce volume; two writers would corrupt both. Updates use the `Recreate`
  strategy, so expect a few seconds of downtime per upgrade.
- **Locked down by default.** uid 1000, read-only root filesystem, no privilege
  escalation, all capabilities dropped, no service-account token.
- **No public default key.** The chart refuses to render without an API key.
- **Your CSVs, read-only.** A PersistentVolumeClaim you provide is mounted at `/data`,
  and SQL cannot read files outside it.

## Install

The chart is in the community repository; it is not yet published to a chart registry.

```bash
git clone https://github.com/tdb-project/tdb-community.git

kubectl create namespace tdb
kubectl -n tdb create secret generic tdb-key \
  --from-literal=api-key="$(openssl rand -hex 32)"

helm install tdb ./tdb-community/charts/tdb-community -n tdb \
  --set apiKey.existingSecret=tdb-key \
  --set data.existingClaim=csv-data
```

`csv-data` is a PVC that holds your CSV files. Without it TDB still starts, with no data
to register.

## Use it

```bash
kubectl -n tdb port-forward svc/tdb-tdb-community 8000:8000
export KEY=$(kubectl -n tdb get secret tdb-key -o jsonpath='{.data.api-key}' | base64 -d)

curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"name":"sales","source_type":"csv","connection":{"file_path":"/data/sales.csv"}}'
```

Point AI tools at `/v1/mcp` on the Service or Ingress; see
[Connect an IDE / AI tool](../getting-started/ide-setup.md).

## Configuration

| Value | Default | Meaning |
|---|---|---|
| `apiKey.existingSecret` / `apiKey.existingSecretKey` | `""` / `api-key` | Secret holding the API key (recommended) |
| `apiKey.value` | `""` | Or a key the chart stores in a Secret. One of the two is required |
| `persistence.size` / `storageClass` / `existingClaim` | `1Gi` / `""` / `""` | Volume for the registry and audit log |
| `data.existingClaim` | `""` | PVC with your CSV files, mounted read-only at `/data` |
| `extraEnv` | `[]` | Any other variable, e.g. `{name: TDB_LEAN_MODE, value: "on"}` |
| `ingress.enabled` / `className` / `host` / `tls` | `false` | Expose it through your ingress controller |
| `resources` | 100m CPU, 256Mi requested; 1Gi limit | |

All variables: [Environment variables](../reference/environment-variables.md).

## Before production

- Serve it over HTTPS: an Ingress with TLS, or a proxy — see
  [HTTPS with Caddy](../recipes/https-caddy.md) for the idea.
- Back up the state volume: it holds your audit trail.
- Read the [production checklist](../recipes/production-checklist.md).

## Enterprise

The enterprise image is licensed and private, and adds settings the community chart does
not have yet: the licence, a pull secret, the PostgreSQL wire-gateway port, and an
encryption key from a Secret. Until it does, ask us for a manifest:
[hello@tdb.jiracorp.co.in](mailto:hello@tdb.jiracorp.co.in).
