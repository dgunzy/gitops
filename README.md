# k0s GitOps Cluster

Production Kubernetes cluster running k0s with Flux Operator APIs as the GitOps control plane.

## Cluster Information

- **Distribution**: k0s v1.34.2 (single-node controller+worker)
- **Server**: Ubuntu 24.04.3 LTS
- **GitOps**: Flux and Flux Operator are both reconciled from Git
- **Repository**: https://github.com/dgunzy/gitops

## Architecture

### Repository Structure

```
├── apps/                               # Application deployments
│   ├── cabot-book/
│   └── cabot-cup/
├── clusters/k0s-cluster-1/            # Cluster entrypoint synced by FluxInstance
│   ├── flux-instance.yaml             # Flux lifecycle (install/upgrade) via FluxOperator API
│   ├── flux-sync.yaml                 # Kustomization dependency graph
│   ├── flux-operator/                 # Flux-managed operator source + kustomization
│   ├── platform/                      # Cluster wrapper to shared platform
│   └── apps/                          # Cluster wrapper to shared apps
└── platform/                          # Shared platform services
    ├── controllers/                   # CRDs and operators
    └── configs/                       # Controller-dependent resources
```

### Platform Components

**Networking & Ingress:**

- **Gateway API v1.2.1**: Modern ingress specification
- **Envoy Gateway v1.6.0**: Gateway API implementation
- **MetalLB v0.15.2**: LoadBalancer for bare metal (L2 mode)
- **LoadBalancer IP**: 38.102.124.212

**TLS Certificates:**

- **cert-manager v1.16.5**: Automatic TLS certificate management
- **Issuer**: Let's Encrypt (production)
- **Integration**: Native Gateway API support (no ingress-shim)

**Secrets Management:**

- **External Secrets Operator v0.20.4**: Sync secrets from external providers
- **Provider**: AWS Secrets Manager (ca-central-1 region)
- **Store**: ClusterSecretStore with AWS credentials

**GitOps:**

- **FluxInstance API**: declarative Flux install and automatic Flux upgrades
- **Flux Operator via Flux**: operator reconciled by Flux from upstream tagged source
- **Sync Chain**: `flux-system` -> `platform-controllers` -> `platform-configs` -> `apps`

## Applications

### cabot-book

- **Domains**: cabotcupbook.com, www.cabotcupbook.com
- **Namespace**: cabot-book
- **Image**: ghcr.io/dgunzy/cabot-book:latest
- **Secrets**: AWS Secrets Manager (`k0s/cabot-book/env`)

### cabot-cup

- **Domain**: cabotcup.ca
- **Namespace**: cabot-cup
- **Image**: ghcr.io/dgunzy/cabot-cup:latest

## Setup & Installation

### Prerequisites

1. **Server Requirements:**

   - Ubuntu 24.04 LTS
   - 2+ CPU cores
   - 4GB+ RAM
   - Public IP address
   - Ports 22, 80, 443, 6443 open

2. **Local Tools:**
   - kubectl
   - flux CLI
   - git
   - SSH access to server

### Initial Cluster Setup

1. **Install k0s:**

   ```bash
   curl -sSLf https://get.k0s.sh | sudo sh
   sudo k0s install controller --single
   sudo k0s start
   ```

2. **Export kubeconfig:**

   ```bash
   sudo k0s kubeconfig admin > ~/.kube/config
   ```

3. **Seed Flux Operator (no Helm):**

   ```bash
   kubectl apply -k "github.com/controlplaneio-fluxcd/flux-operator/config/default?ref=v0.42.1"
   ```

4. **Apply FluxInstance (operator installs Flux + syncs this repo):**

   ```bash
   kubectl apply -f clusters/k0s-cluster-1/flux-instance.yaml
   kubectl -n flux-system wait fluxinstance/flux --for=condition=Ready --timeout=10m
   ```

5. **Create AWS credentials secret:**
   ```bash
   kubectl create secret generic aws-credentials \\
     --namespace external-secrets \\
     --from-literal=access-key-id="<AWS_ACCESS_KEY>" \\
     --from-literal=secret-access-key="<AWS_SECRET_KEY>"
   ```

6. **Verify GitOps graph is active:**

   ```bash
   flux get source git -n flux-system
   flux get kustomization -n flux-system
   ```

### DNS Configuration

Point your domains to the LoadBalancer IP:

```
A     cabotcup.ca              → 38.102.124.212
A     cabotcupbook.com         → 38.102.124.212
CNAME www.cabotcupbook.com     → cabotcupbook.com
```

## Troubleshooting

### DNS Issues in Cluster

If cluster DNS stops working (pods can't resolve internal/external names):

**Cause**: iptables NAT rules can be accidentally flushed, breaking Kubernetes networking.

**Fix**: Restart k0s to rebuild iptables rules:

```bash
ssh root@<server> "systemctl restart k0scontroller"
```

### Flux Can't Reach GitHub

Check if DNS is working from within the cluster:

```bash
kubectl run test-dns --image=busybox:1.28 --restart=Never -- sleep 3600
kubectl exec test-dns -- nslookup github.com
kubectl delete pod test-dns
```

### Certificate Issues

View certificate status:

```bash
kubectl get certificate -n envoy-gateway-system
kubectl describe certificate <cert-name> -n envoy-gateway-system
```

Check ACME challenges:

```bash
kubectl get challenges -A
kubectl logs -n cert-manager deploy/cert-manager --tail=50
```

### Debugging with Flux CLI

```bash
# Check all Flux resources
flux get all -A

# Force reconciliation
flux reconcile source git flux-system
flux reconcile source git flux-operator -n flux-system
flux reconcile kustomization flux-operator -n flux-system
flux reconcile kustomization platform-controllers
flux reconcile kustomization platform-configs
flux reconcile kustomization apps

# Use amend workflow for rapid iteration
git add -A && amend
flux reconcile source git flux-system
```

## Important Considerations

1. **iptables Rules**: Never run `iptables -t nat -F PREROUTING` or similar flush commands as they break Kubernetes networking. If you need to modify rules, restart k0s afterwards.

2. **Flux Dependency Chain**: Resources are applied in order:

   - `platform-controllers` (CRDs & operators)
   - `platform-configs` (resources requiring CRDs)
   - `apps` (application workloads)

3. **MetalLB**: Required for LoadBalancer support on bare metal. Uses L2 advertisement to make the server IP available to services.

4. **Gateway API**: Using Gateway API v1 (GA) instead of deprecated Ingress. All routes defined as HTTPRoute resources.

5. **External Secrets**: Secrets are never stored in Git. They're synced from AWS Secrets Manager using the External Secrets Operator.

6. **Certificate Management**: cert-manager automatically requests and renews TLS certificates from Let's Encrypt using HTTP-01 challenges through Gateway API HTTPRoutes.

## Maintenance

### Updating Applications

Applications automatically update when new images are pushed (if using image automation) or when manifests are updated in Git and synced by Flux.

### Updating Platform Components

Update Helm chart versions in `platform/controllers/*/helmrelease.yaml` and commit. Flux will automatically reconcile.

### Updating Flux and Flux Operator

- **Flux controllers**: tune semver in `clusters/k0s-cluster-1/flux-instance.yaml` (`spec.distribution.version`, e.g. `2.x` or `2.8.x`).
- **Flux Operator**: tune semver in `clusters/k0s-cluster-1/flux-operator/source.yaml` (`spec.ref.semver`, currently `0.42.x`).
- Commit and push; Flux applies upgrades.

### Backup & Recovery

Critical data to backup:

- AWS Secrets Manager secrets
- Application persistent volumes (if any)
- k0s etcd backup: `sudo k0s backup --save-path=/root/k0s-backup.tar.gz`

## Resources

- [k0s Documentation](https://docs.k0sproject.io/)
- [Flux Documentation](https://fluxcd.io/flux/)
- [Flux Operator Docs](https://fluxoperator.dev/docs/)
- [Flux Operator Source](https://github.com/controlplaneio-fluxcd/flux-operator)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [cert-manager](https://cert-manager.io/)
- [External Secrets Operator](https://external-secrets.io/)
- [MetalLB](https://metallb.universe.tf/)
