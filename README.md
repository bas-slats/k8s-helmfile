# Homelab Kubernetes Infrastructure

Helmfile-based infrastructure deployment for a homelab Kubernetes cluster.

## Prerequisites

### k3s Installation

Install k3s **without** default Flannel CNI (we use Cilium instead):

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy --disable=traefik" sh -
```

### Required CLI Tools

- `kubectl` - Kubernetes CLI
- `helm` - Helm package manager
- `helmfile` - Declarative Helm deployment

### macOS (Homebrew)

```bash
brew install kubectl helm helmfile
```

## Directory Structure

```
helmfile/
├── helmfile.yaml           # Main helmfile configuration
├── environments/
│   └── default.yaml        # Environment-specific values
├── releases/               # Individual release configurations
│   ├── 01-cilium.yaml
│   ├── 02-metallb.yaml
│   └── ...
├── values/                 # Helm values for each release
│   ├── cilium.yaml
│   ├── metallb.yaml
│   └── ...
└── manifests/              # Kubernetes manifests applied via hooks
    ├── metallb-config.yaml
    ├── cert-manager-issuers.yaml
    └── ...
```

## Deployment Order

1. **Cluster Foundation** - Cilium (CNI), MetalLB (LoadBalancer)
2. **Storage** - Longhorn (distributed storage)
3. **Certificates** - cert-manager (certificate automation)
4. **Secrets** - Vault, External Secrets Operator
5. **GitOps** - ArgoCD, ChartMuseum
6. **Service Mesh** - Linkerd (CRDs, Control Plane, Viz)
7. **Observability** - OTel Operator/Collector, Tempo, Pyroscope, Loki, Mimir, Grafana
8. **Ingress** - Cloudflared (Cloudflare Tunnel)
9. **Utilities** - Reloader, Kured

## Quick Start

### 1. Configure Environment

Edit `environments/default.yaml` to match your network:

```yaml
network:
  loadBalancerRange: "192.168.1.200-192.168.1.220"  # Your LAN range
```

### 2. Deploy Infrastructure

```bash
cd helmfile
helmfile sync
```

### 3. Initialize Vault

After deployment, Vault needs to be initialized:

```bash
# Get Vault pod
kubectl exec -it vault-0 -n vault -- vault operator init

# Save the unseal keys and root token securely!

# Unseal Vault (repeat 3 times with different keys)
kubectl exec -it vault-0 -n vault -- vault operator unseal

# Update the External Secrets vault token
kubectl patch secret vault-token -n external-secrets \
  --type='json' \
  -p='[{"op":"replace","path":"/stringData/token","value":"YOUR_ROOT_TOKEN"}]'
```

### 4. Configure Cloudflare Tunnel

1. Create a tunnel in Cloudflare Zero Trust dashboard
2. Copy the tunnel token
3. Store in Vault:

```bash
kubectl exec -it vault-0 -n vault -- vault kv put secret/cloudflare/tunnel token="YOUR_TUNNEL_TOKEN"
```

## Accessing Services

### Grafana

```bash
kubectl port-forward svc/grafana 3000:80 -n grafana
# Open http://localhost:3000
# Default: admin / admin (change on first login)
```

### ArgoCD

```bash
kubectl port-forward svc/argocd-server 8080:443 -n argocd
# Open https://localhost:8080
# Get password: kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
```

### Vault UI

```bash
kubectl port-forward svc/vault 8200:8200 -n vault
# Open http://localhost:8200
```

### Linkerd Dashboard

```bash
kubectl port-forward svc/web 8084:8084 -n linkerd-viz
# Open http://localhost:8084
```

## Observability Stack

| Component | Purpose | Port |
|-----------|---------|------|
| Grafana | Unified UI | 3000 |
| Mimir | Metrics storage | 80 (nginx) |
| Loki | Log aggregation | 3100 |
| Tempo | Distributed tracing | 3100 |
| Pyroscope | Continuous profiling | 4040 |
| OTel Collector | Telemetry pipeline | 4317 (gRPC), 4318 (HTTP) |

### Sending Telemetry

Configure your applications to send OTLP data to:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.otel.svc.cluster.local:4317
```

## Customization

### Adding a New Release

1. Create `releases/XX-myapp.yaml`
2. Create `values/myapp.yaml`
3. Add to `helmfile.yaml` under `helmfiles:`

### Changing Resource Limits

Edit the appropriate values file in `values/` directory.

### Adding Secrets

1. Store secret in Vault
2. Create ExternalSecret in `manifests/` to sync to Kubernetes

## Troubleshooting

### Helmfile Sync Fails

```bash
# Check pending releases
helm list -A --pending

# Rollback if needed
helm rollback <release> <revision> -n <namespace>
```

### Cilium Issues

```bash
# Check Cilium status
kubectl exec -it -n kube-system ds/cilium -- cilium status
```

### Storage Issues

```bash
# Check Longhorn
kubectl get volumes -n longhorn-system
```

## Maintenance

### Update Helm Repos

```bash
helmfile repos
```

### Upgrade All Releases

```bash
helmfile sync
```

### Backup Vault

```bash
kubectl exec -it vault-0 -n vault -- vault operator raft snapshot save /tmp/vault-backup.snap
kubectl cp vault/vault-0:/tmp/vault-backup.snap ./vault-backup.snap
```
