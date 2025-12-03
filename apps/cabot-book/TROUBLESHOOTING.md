# Troubleshooting Turso Connectivity Issues

## Problem
The cabot-book application is experiencing connectivity issues connecting to Turso database:
```
Error creating connector: failed to sync database
error code = 1: Error syncing database: replication error: Timeout performing handshake with primary
```

## Root Cause
When `networkPolicy: true` is set in FluxInstance, Kubernetes NetworkPolicies are enabled cluster-wide. By default, pods are isolated and cannot make outbound connections unless explicitly allowed.

## Solution Applied

### 1. NetworkPolicy Created
A NetworkPolicy has been added to allow:
- DNS resolution (port 53 UDP/TCP to kube-dns)
- HTTPS outbound (port 443) for Turso/libsql.cloud
- HTTP outbound (port 80) for Let's Encrypt challenges

### 2. Diagnostic Steps

#### Test DNS Resolution
```bash
# Create a debug pod
kubectl apply -f apps/cabot-book/debug-pod.yaml

# Wait for it to be ready
kubectl wait --for=condition=Ready pod/network-debug -n cabot-book

# Test DNS resolution for Turso
kubectl exec -it network-debug -n cabot-book -- nslookup libsql.cloud
kubectl exec -it network-debug -n cabot-book -- nslookup <your-turso-instance>.turso.io
```

#### Test HTTPS Connectivity
```bash
# Test HTTPS connection to Turso
kubectl exec -it network-debug -n cabot-book -- curl -v https://libsql.cloud
kubectl exec -it network-debug -n cabot-book -- curl -v https://<your-turso-instance>.turso.io
```

#### Check NetworkPolicy Status
```bash
# Verify NetworkPolicy is applied
kubectl get networkpolicy -n cabot-book

# Describe the policy
kubectl describe networkpolicy allow-egress -n cabot-book
```

#### Test from Application Pod
```bash
# Get the pod name
POD=$(kubectl get pods -n cabot-book -l app=cabot-book -o jsonpath='{.items[0].metadata.name}')

# Test DNS
kubectl exec -it $POD -n cabot-book -- nslookup libsql.cloud

# Test HTTPS
kubectl exec -it $POD -n cabot-book -- curl -v https://libsql.cloud
```

### 3. Verify Turso Connection String

Check that the Turso connection string in AWS Secrets Manager (`k0s/cabot-book/env`) is correct:
- Format: `libsql://<database-name>-<org-name>.turso.io`
- Should include auth token
- Check if using primary or replica URL

### 4. Common Issues

#### DNS Not Resolving
If DNS fails, check CoreDNS:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

#### NetworkPolicy Not Applied
If NetworkPolicy isn't working:
```bash
# Check if NetworkPolicy controller is running
kubectl get pods -n kube-system | grep network-policy

# Verify pod labels match NetworkPolicy selector
kubectl get pods -n cabot-book -l app=cabot-book --show-labels
```

#### Firewall Blocking Outbound
Check if the node firewall is blocking outbound connections:
```bash
# SSH to the node and test
ssh root@<node-ip>
curl -v https://libsql.cloud
```

### 5. Alternative: Disable NetworkPolicy (Not Recommended)

If NetworkPolicies are causing too many issues, you can disable them:
```yaml
# In clusters/my-cluster/flux-instance.yaml
cluster:
  networkPolicy: false  # Change from true to false
```

**Warning**: This reduces security isolation between pods.

## Next Steps

1. Apply the NetworkPolicy: `kubectl apply -f apps/cabot-book/network-policy.yaml`
2. Restart the cabot-book pod: `kubectl delete pod -n cabot-book -l app=cabot-book`
3. Monitor logs: `kubectl logs -f -n cabot-book -l app=cabot-book`
4. Run diagnostics using the commands above

