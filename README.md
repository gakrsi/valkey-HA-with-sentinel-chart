# Valkey Sentinel High Availability Helm Chart

A Helm chart for deploying Valkey with Sentinel-based High Availability on Kubernetes.

## Overview

This Helm chart deploys a highly available Valkey cluster with automatic failover using Sentinel. The deployment includes:

- **Valkey StatefulSet**: 3 replicas (1 master, 2 replicas) by default
- **Sentinel Deployment**: 3 sentinel instances for monitoring and automatic failover
- **Persistent Storage**: Configurable persistent volumes for data durability
- **PodSecurity Compliance**: Restricted security context following Kubernetes best practices

## Architecture

```
                    Applications
                         │
                         │
                         ▼
               valkey-sentinel:26379
                         │
          ┌──────────────┴──────────────┐
          │ Sentinel quorum (3 pods)    │
          │ election + failover         │
          └──────────────┬──────────────┘
                         │
                         ▼
                  Valkey Master
                  (StatefulSet)
                     valkey-0
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
          valkey-1               valkey-2
          Replica                Replica
```

## Prerequisites

- Kubernetes cluster (1.19+)
- kubectl configured
- Helm 3.x
- StorageClass available for persistent volumes
- (Optional) PodSecurity admission controller for restricted policy

## Installation

### Quick Start

```bash
# Add the helm chart repository (if published)
# helm repo add valkey-ha <repository-url>
# helm repo update

# Install from local directory
helm install my-valkey-ha . --namespace valkey-ha --create-namespace

# Or with custom values
helm install my-valkey-ha . --namespace valkey-ha --create-namespace -f custom-values.yaml
```

### Verify Installation

```bash
# Check pod status
kubectl get pods -n valkey-ha

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=valkey-sentinel-ha -n valkey-ha --timeout=300s

# Verify Sentinel is monitoring the master
kubectl exec -it my-valkey-ha-0 -n valkey-ha -- \
  valkey-cli -h my-valkey-ha-sentinel -p 26379 \
  sentinel get-master-addr-by-name mymaster
```

## Configuration

### Key Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `auth.enabled` | Enable password authentication | `true` |
| `auth.password` | Valkey password (leave empty for auto-generated) | `""` |
| `auth.existingSecret` | Use existing secret for password | `""` |
| `valkey.replicas` | Number of Valkey replicas | `3` |
| `valkey.image.repository` | Valkey image repository | `valkey/valkey` |
| `valkey.image.tag` | Valkey image tag | `7.2-alpine` |
| `valkey.service.port` | Valkey service port | `6379` |
| `valkey.persistence.enabled` | Enable persistent storage | `true` |
| `valkey.persistence.size` | Size of persistent volume | `2Gi` |
| `valkey.persistence.storageClass` | Storage class name | `""` (default) |
| `sentinel.replicas` | Number of Sentinel replicas | `3` |
| `sentinel.service.port` | Sentinel service port | `26379` |
| `sentinel.masterName` | Name of the master to monitor | `mymaster` |
| `sentinel.quorum` | Minimum sentinels for master election | `2` |
| `sentinel.downAfterMilliseconds` | Time before declaring master down | `5000` |
| `sentinel.failoverTimeout` | Failover timeout in milliseconds | `10000` |

### Example: Custom Values

Create a `custom-values.yaml` file:

```yaml
valkey:
  replicas: 5
  persistence:
    size: 10Gi
    storageClass: fast-ssd
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

sentinel:
  replicas: 3
  quorum: 2
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi
```

Install with custom values:

```bash
helm install my-valkey-ha . -f custom-values.yaml --namespace valkey-ha --create-namespace
```

## Authentication

By default, password authentication is **enabled** for security. The chart provides three ways to configure authentication:

### Auto-generated Password (Default)

When `auth.enabled` is `true` and `auth.password` is empty, a random 32-character password is automatically generated:

```yaml
auth:
  enabled: true
  password: ""  # Auto-generate
```

Retrieve the password after installation:

```bash
export VALKEY_PASSWORD=$(kubectl get secret --namespace valkey-ha my-valkey-ha-auth -o jsonpath="{.data.password}" | base64 -d)
echo "Password: $VALKEY_PASSWORD"
```

### Custom Password

Specify your own password in values.yaml:

```yaml
auth:
  enabled: true
  password: "your-strong-password-here"
```

### Existing Secret

Use an existing Kubernetes secret:

```bash
# Create a secret first
kubectl create secret generic my-valkey-secret \
  --from-literal=password=my-custom-password \
  --namespace valkey-ha
```

```yaml
auth:
  enabled: true
  existingSecret: "my-valkey-secret"
```

### Disable Authentication (Not Recommended)

For development/testing only:

```yaml
auth:
  enabled: false
```

## Usage

### Connecting to Valkey

Applications should connect to Valkey through Sentinel for automatic failover:

```
Service: <release-name>-sentinel.<namespace>.svc.cluster.local
Port: 26379
Master Name: mymaster
```

### Example Application Configuration

#### Python (redis-py with sentinel)

```python
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('<release-name>-sentinel.<namespace>.svc.cluster.local', 26379)
], socket_timeout=0.1)

# If authentication is enabled, provide password
master = sentinel.master_for(
    'mymaster',
    socket_timeout=0.1,
    password='your-valkey-password'
)
slave = sentinel.slave_for(
    'mymaster',
    socket_timeout=0.1,
    password='your-valkey-password'
)

# Write to master
master.set('key', 'value')

# Read from slave
value = slave.get('key')
```

#### Node.js (ioredis)

```javascript
const Redis = require('ioredis');

const redis = new Redis({
  sentinels: [
    { host: '<release-name>-sentinel.<namespace>.svc.cluster.local', port: 26379 }
  ],
  name: 'mymaster',
  // If authentication is enabled, provide password
  password: 'your-valkey-password'
});

redis.set('key', 'value');
redis.get('key', (err, result) => {
  console.log(result);
});
```

### Testing Failover

To test automatic failover:

```bash
# Delete the master pod
kubectl delete pod <release-name>-0 -n <namespace>

# Sentinel will automatically promote a replica to master
# Check the new master
kubectl exec -it <release-name>-1 -n <namespace> -- \
  valkey-cli -h <release-name>-sentinel -p 26379 \
  sentinel get-master-addr-by-name mymaster
```

## Upgrading

```bash
# Upgrade with new values
helm upgrade my-valkey-ha . --namespace valkey-ha -f custom-values.yaml

# View upgrade history
helm history my-valkey-ha --namespace valkey-ha
```

## Uninstalling

```bash
# Uninstall the release
helm uninstall my-valkey-ha --namespace valkey-ha

# Delete PVCs (if you want to remove data)
kubectl delete pvc -n valkey-ha -l app.kubernetes.io/name=valkey-sentinel-ha
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl describe pod <pod-name> -n <namespace>

# Check logs
kubectl logs <pod-name> -n <namespace>
```

### Sentinel Not Detecting Master

```bash
# Check Sentinel logs
kubectl logs <sentinel-pod-name> -n <namespace>

# Verify Sentinel configuration
kubectl exec -it <sentinel-pod-name> -n <namespace> -- \
  valkey-cli -p 26379 info sentinel
```

### Storage Issues

```bash
# Check PVC status
kubectl get pvc -n <namespace>

# Check PV status
kubectl get pv
```

## Security Considerations

This chart implements security best practices:

- **Password authentication**: Enabled by default with auto-generated secure passwords
- **Non-root user**: Containers run as user 1001
- **Drop capabilities**: All capabilities are dropped
- **No privilege escalation**: Privilege escalation is prevented
- **Seccomp profile**: Runtime default seccomp profile is applied
- **Read-only root filesystem**: (Optional) Can be enabled via values
- **Secret management**: Passwords stored in Kubernetes Secrets, not ConfigMaps

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This Helm chart is open source and available under the [MIT License](LICENSE).

## References

- [Valkey Documentation](https://valkey.io/)
- [Redis Sentinel Documentation](https://redis.io/docs/management/sentinel/)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
