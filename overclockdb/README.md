# OverclockDB Helm Chart

A Helm chart for deploying OverclockDB - a high-performance, in-memory text search database written in Rust.

## Architecture

This chart deploys OverclockDB as a **StatefulSet** (not Deployment) because:

- **Stable pod identity**: Pods get predictable names (`overclockdb-0`, `overclockdb-1`)
- **Per-pod storage**: Each replica gets its own PVC via `volumeClaimTemplates`
- **Ordered operations**: Graceful scaling and termination for safe WAL flushing
- **Future-ready**: Prepared for clustering if added later

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support (if persistence is enabled)

## Installation

```bash
# Install from local directory
helm install overclockdb ./helm/overclockdb

# Install with custom values
helm install overclockdb ./helm/overclockdb -f custom-values.yaml

# Install in a specific namespace
helm install overclockdb ./helm/overclockdb -n overclockdb --create-namespace
```

## Uninstallation

```bash
helm uninstall overclockdb

# Note: PVCs are NOT automatically deleted (data protection)
# To delete data:
kubectl delete pvc -l app.kubernetes.io/name=overclockdb
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Image repository | `overclockdb` |
| `image.tag` | Image tag | `""` (uses appVersion) |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `8108` |
| `overclockdb.port` | Application port | `8108` |
| `overclockdb.dataDir` | Data directory path | `/data` |
| `overclockdb.persistence` | Enable data persistence | `true` |
| `overclockdb.logLevel` | Log level (RUST_LOG) | `info` |
| `persistence.enabled` | Enable PVC | `true` |
| `persistence.storageClass` | Storage class | `""` |
| `persistence.size` | PVC size per pod | `10Gi` |
| `persistence.accessMode` | PVC access mode | `ReadWriteOnce` |
| `postgres.enabled` | Enable PostgreSQL sync | `false` |
| `postgres.url` | PostgreSQL connection URL | `""` |
| `postgres.table` | Source table name | `""` |
| `postgres.collection` | Target collection name | `""` |
| `resources.requests.memory` | Memory request | `512Mi` |
| `resources.requests.cpu` | CPU request | `500m` |
| `resources.limits.memory` | Memory limit | `2Gi` |
| `resources.limits.cpu` | CPU limit | `2000m` |

## Examples

### Basic Installation

```bash
helm install overclockdb ./helm/overclockdb
```

### With Custom Resources

```yaml
# custom-values.yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "4000m"

persistence:
  size: 50Gi
```

### With PostgreSQL Sync

```yaml
# postgres-sync-values.yaml
postgres:
  enabled: true
  url: "postgres://user:password@postgres:5432/mydb"
  table: "products"
  collection: "products"
```

For production, use an existing secret:

```yaml
postgres:
  enabled: true
  existingSecret: "my-postgres-secret"
  existingSecretKey: "connection-url"
  table: "products"
  collection: "products"
```

### LoadBalancer Service

```yaml
# loadbalancer-values.yaml
service:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

## Services

The chart creates two services:

1. **Headless service** (`overclockdb-headless`): Required for StatefulSet pod DNS
2. **Client service** (`overclockdb`): For application access (ClusterIP/LoadBalancer)

## Health Checks

The chart configures liveness and readiness probes:

- **Liveness**: `GET /health` - Basic health check
- **Readiness**: `GET /ready` - Ready to accept traffic

## Persistence

The StatefulSet uses `volumeClaimTemplates` to create a PVC for each pod. The data directory contains:

- `wal/` - Write-Ahead Logs for crash recovery
- `snapshots/` - LZ4-compressed snapshots

PVC naming convention: `data-overclockdb-0`, `data-overclockdb-1`, etc.

**Important**: PVCs are retained after uninstall to protect data. Delete manually if needed.

## Security

The chart runs with security best practices:

- Non-root user (UID 1000)
- Dropped capabilities
- No privilege escalation

## Troubleshooting

```bash
# Check StatefulSet status
kubectl get statefulset overclockdb

# Check pod status
kubectl get pods -l app.kubernetes.io/name=overclockdb

# Check PVCs
kubectl get pvc -l app.kubernetes.io/name=overclockdb

# View logs
kubectl logs overclockdb-0

# Check health
kubectl exec -it overclockdb-0 -- curl localhost:8108/health

# Port forward for local access
kubectl port-forward svc/overclockdb 8108:8108
```
