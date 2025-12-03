# Migration Comparison: ingress-nginx → Gateway API + Envoy Gateway + MetalLB

## Overview

This document explains the differences between the old **ingress-nginx** setup and the new **Gateway API + Envoy Gateway + MetalLB** architecture, including unintended changes and architectural differences.

---

## Architecture Comparison

### Old Setup: ingress-nginx

```
Internet → NodePort/ExternalIP (38.102.124.212) → ingress-nginx Service → Ingress Resources → App Services
```

**Components:**
- **ingress-nginx**: Traditional Ingress controller (NGINX-based)
- **Ingress Resources**: Kubernetes Ingress API (`networking.k8s.io/v1`)
- **Service Type**: NodePort or ExternalIPs configured directly on ingress-nginx service
- **TLS**: cert-manager with ingress-shim (automatic certificate injection into Ingress)

### New Setup: Gateway API + Envoy Gateway + MetalLB

```
Internet → MetalLB LoadBalancer (38.102.124.212) → Envoy Gateway Service → Gateway Resource → HTTPRoute Resources → App Services
```

**Components:**
- **Gateway API**: Modern Kubernetes networking standard (`gateway.networking.k8s.io/v1`)
- **Envoy Gateway**: Gateway API implementation (Envoy proxy)
- **MetalLB**: LoadBalancer implementation for bare metal
- **Gateway Resource**: Defines listeners and TLS termination
- **HTTPRoute Resources**: Define routing rules (replaces Ingress)
- **TLS**: cert-manager with native Gateway API support (no ingress-shim)

---

## Key Differences

### 1. **Load Balancing Architecture**

#### Old (ingress-nginx):
- **Service Type**: NodePort or ExternalIPs
- **IP Assignment**: Static IP configured directly on ingress-nginx service via `externalIPs: ["38.102.124.212"]`
- **How it worked**: Traffic hit the node IP directly, routed to NodePort or ExternalIP

#### New (MetalLB + Envoy Gateway):
- **Service Type**: LoadBalancer
- **IP Assignment**: MetalLB manages IP allocation via IPAddressPool
- **How it works**: 
  - MetalLB advertises the IP (38.102.124.212) via L2 (ARP)
  - Envoy Gateway service gets LoadBalancer IP from MetalLB
  - Traffic routes through standard LoadBalancer mechanism

**Unintended Change**: 
- **More complex**: Requires MetalLB operator + IPAddressPool + L2Advertisement
- **Better for multi-node**: MetalLB can distribute across nodes (though you're single-node)
- **More "cloud-native"**: Matches cloud provider LoadBalancer behavior

---

### 2. **Routing Configuration**

#### Old (Ingress Resources):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cabot-book
  namespace: cabot-book
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - cabotcupbook.com
        - www.cabotcupbook.com
      secretName: cabotcupbook-tls
  rules:
    - host: cabotcupbook.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: cabot-book-service
                port: 7070
```

**Characteristics:**
- Single resource per app
- TLS configuration embedded in Ingress
- Annotations for ingress class and cert-manager
- Simple path-based routing

#### New (Gateway + HTTPRoute):
```yaml
# Gateway (shared infrastructure)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  listeners:
    - name: https-cabotcupbook
      hostname: "cabotcupbook.com"
      protocol: HTTPS
      port: 443
      tls:
        certificateRefs:
          - name: cabotcupbook-cert

# HTTPRoute (per app)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  parentRefs:
    - name: main-gateway
      sectionName: https-cabotcupbook
  hostnames:
    - "cabotcupbook.com"
  rules:
    - backendRefs:
        - name: cabot-book-service
          port: 8080
```

**Characteristics:**
- **Two resources**: Gateway (infrastructure) + HTTPRoute (routing)
- **Separation of concerns**: TLS/listeners in Gateway, routing in HTTPRoute
- **Hostname-based listeners**: Each domain gets its own listener
- **More explicit**: No annotations, everything in spec

**Unintended Changes:**
- **More verbose**: Requires Gateway + HTTPRoute instead of single Ingress
- **Hostname per listener**: Each domain needs separate listener (https-cabotcupbook, https-www-cabotcupbook, https-cabotcup)
- **Port change**: Service port changed from 7070 → 8080 (this was a bug fix, not architectural)

---

### 3. **TLS Certificate Management**

#### Old (cert-manager + ingress-shim):
```yaml
# ClusterIssuer
spec:
  solvers:
    - http01:
        ingress:
          class: nginx

# Ingress (automatic cert injection via annotation)
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

**How it worked:**
- cert-manager ingress-shim watches Ingress resources
- Sees annotation → creates Certificate → injects Secret into Ingress
- ingress-nginx reads Secret automatically

#### New (cert-manager + Gateway API):
```yaml
# ClusterIssuer
spec:
  solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
            - name: main-gateway
              namespace: envoy-gateway-system

# Gateway (explicit certificate reference)
spec:
  listeners:
    - name: https-cabotcupbook
      tls:
        certificateRefs:
          - name: cabotcupbook-cert  # cert-manager creates this
```

**How it works:**
- cert-manager watches Gateway resources (native Gateway API support)
- Creates Certificate → creates Secret → Gateway references Secret
- More explicit certificate lifecycle

**Unintended Changes:**
- **Certificate names**: Must match listener names (cabotcupbook-cert, cabotcup-cert)
- **Gateway annotation**: Still uses `cert-manager.io/cluster-issuer` annotation on Gateway
- **No automatic injection**: Must explicitly reference certificates in Gateway spec

---

### 4. **Service Port Configuration**

#### Old:
```yaml
# Service
spec:
  ports:
    - port: 7070
      targetPort: web

# Ingress
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                port: 7070
```

#### New:
```yaml
# Service
spec:
  ports:
    - port: 8080
      targetPort: web

# HTTPRoute
spec:
  rules:
    - backendRefs:
        - port: 8080
```

**Unintended Change**: 
- Port changed from 7070 → 8080
- This was actually a **bug fix** - container exposes 8080, service should match
- Old setup had mismatch (service port 7070, container port 8080)

---

### 5. **Multiple Domains Handling**

#### Old (Single Ingress):
```yaml
# One Ingress handles multiple hosts
spec:
  tls:
    - hosts:
        - cabotcupbook.com
        - www.cabotcupbook.com
  rules:
    - host: cabotcupbook.com
    - host: www.cabotcupbook.com
```

#### New (Separate Listeners):
```yaml
# Gateway has separate listeners per hostname
spec:
  listeners:
    - name: https-cabotcupbook
      hostname: "cabotcupbook.com"
    - name: https-www-cabotcupbook
      hostname: "www.cabotcupbook.com"

# HTTPRoute references both
spec:
  parentRefs:
    - sectionName: https-cabotcupbook
    - sectionName: https-www-cabotcupbook
  hostnames:
    - "cabotcupbook.com"
    - "www.cabotcupbook.com"
```

**Unintended Change**:
- **More verbose**: Each hostname needs separate listener
- **HTTPRoute must reference all**: Must list all parentRefs for each listener
- **Certificate sharing**: Both listeners can share same certificate (cabotcupbook-cert)

---

### 6. **HTTP to HTTPS Redirect**

#### Old (ingress-nginx):
- Automatic redirect via annotation: `nginx.ingress.kubernetes.io/ssl-redirect: "true"`
- Or configured in ingress-nginx ConfigMap

#### New (Envoy Gateway):
- **No automatic redirect**: HTTP and HTTPS are separate listeners
- HTTPRoute can attach to HTTP listener, but no automatic redirect
- **Missing feature**: Need to configure HTTP→HTTPS redirect manually (not currently configured)

**Unintended Change**: 
- **HTTP works but doesn't redirect**: Currently HTTP listener exists but doesn't redirect to HTTPS
- **Security concern**: Users can access via HTTP (port 80) without redirect

---

### 7. **Network Policies**

#### Old:
- NetworkPolicies may not have been configured
- ingress-nginx pods could make outbound connections freely

#### New:
- **NetworkPolicy enabled**: `networkPolicy: true` in FluxInstance
- **Pods isolated by default**: All pods blocked from egress unless NetworkPolicy allows
- **Required NetworkPolicy**: Created `allow-egress` policy for cabot-book to allow DNS, HTTP, HTTPS

**Unintended Change**:
- **Stricter security**: Pods can't make outbound connections without explicit policy
- **More configuration**: Must create NetworkPolicy for each app that needs outbound access
- **Turso connectivity issue**: App couldn't connect to Turso until NetworkPolicy was added

---

## Summary of Unintended Changes

1. **More Complex Setup**
   - Requires MetalLB + Gateway + HTTPRoute (vs single Ingress)
   - More YAML files to manage

2. **Hostname-Based Listeners**
   - Each domain needs separate listener in Gateway
   - HTTPRoute must reference all relevant listeners

3. **No Automatic HTTP→HTTPS Redirect**
   - HTTP listener exists but doesn't redirect
   - Security concern: HTTP traffic not automatically secured

4. **Stricter Network Policies**
   - Pods isolated by default
   - Must create NetworkPolicy for outbound access

5. **Port Mismatch Fixed**
   - Service port corrected from 7070 → 8080
   - This was actually fixing a bug, not an unintended change

6. **Certificate Management**
   - More explicit certificate references
   - Certificate names must match listener names

---

## Recommendations

### 1. **Add HTTP→HTTPS Redirect**
Configure Envoy Gateway to redirect HTTP to HTTPS. This requires either:
- HTTPRoute with redirect filter (Gateway API feature)
- Or configure Envoy Gateway to handle redirects

### 2. **Simplify Gateway Listeners**
Consider using wildcard listeners or single HTTPS listener with multiple hostnames if Gateway API supports it.

### 3. **Document NetworkPolicy Requirements**
Add NetworkPolicy to each app namespace that needs outbound access.

### 4. **Consider Ingress Compatibility**
If simplicity is preferred, could use Gateway API with ingress-nginx (it supports Gateway API), but Envoy Gateway is more modern.

---

## Benefits of New Architecture

Despite the complexity, the new setup provides:

1. **Standard Compliance**: Gateway API is the future Kubernetes standard
2. **Better Separation**: Infrastructure (Gateway) vs Application (HTTPRoute)
3. **More Flexible**: Gateway API supports advanced routing features
4. **Cloud-Native**: MetalLB provides LoadBalancer like cloud providers
5. **Better Security**: NetworkPolicies enforce network isolation
6. **Modern Stack**: Envoy Gateway is actively developed, ingress-nginx is maintenance mode

