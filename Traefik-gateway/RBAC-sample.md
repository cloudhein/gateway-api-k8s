# Gateway API RBAC Security Guide

## Understanding Gateway API Security Model

Gateway API follows a **multi-role security model** where different personas have different levels of access to resources.

### The 3 Core Resources & Roles

| Resource | What It Does | Who Manages It |
|----------|--------------|----------------|
| **GatewayClass** | Defines gateway template/configuration | Infrastructure Provider |
| **Gateway** | Creates actual gateway instance | Cluster Operator |
| **HTTPRoute/TCPRoute** | Defines traffic routing rules | Application Developer |

### The 3 Primary Roles

1. **Ian (Infrastructure Provider)** - Platform/Cloud admin
2. **Chihiro (Cluster Operator)** - Kubernetes cluster admin  
3. **Ana (Application Developer)** - App teams

## RBAC Permission Matrix

### Simple 3-Tier Model
| Role | GatewayClass | Gateway | Routes |
|------|-------------|---------|---------|
| Infrastructure Provider | ✅ Read/Write | ✅ Read/Write | ✅ Read/Write |
| Cluster Operator | ❌ Read Only | ✅ Read/Write | ✅ Read/Write |
| Application Developer | ❌ Read Only | ❌ Read Only | ✅ Read/Write |

## Practical RBAC Configuration Examples

### 1. Infrastructure Provider (Full Admin)

```yaml
# Infrastructure Provider - Full access to all Gateway API resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gateway-infrastructure-admin
rules:
# GatewayClass management
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gatewayclasses"]
  verbs: ["*"]
# Gateway management across all namespaces
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gateways"]
  verbs: ["*"]
# Route management across all namespaces
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["httproutes", "tcproutes", "grpcroutes", "tlsroutes"]
  verbs: ["*"]
# Status updates for all resources
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["*/status"]
  verbs: ["update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ian-infrastructure-admin
subjects:
- kind: User
  name: ian@company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: gateway-infrastructure-admin
  apiGroup: rbac.authorization.k8s.io
```

### 2. Cluster Operator (Gateway & Route Management)

```yaml
# Cluster Operator - Gateway and Route management, GatewayClass read-only
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gateway-cluster-operator
rules:
# Read-only access to GatewayClasses
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gatewayclasses"]
  verbs: ["get", "list", "watch"]
# Full Gateway management
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gateways"]
  verbs: ["*"]
# Full Route management
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["httproutes", "tcproutes", "grpcroutes", "tlsroutes"]
  verbs: ["*"]
# ReferenceGrant management (for cross-namespace references)
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["referencegrants"]
  verbs: ["*"]
# Status updates
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["*/status"]
  verbs: ["update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: chihiro-cluster-operator
subjects:
- kind: User
  name: chihiro@company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: gateway-cluster-operator
  apiGroup: rbac.authorization.k8s.io
```

### 3. Application Developer (Route Management Only)

```yaml
# Application Developer - Route management in specific namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production  # Specific namespace
  name: gateway-app-developer
rules:
# Read-only access to Gateways (to know what's available)
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gateways"]
  verbs: ["get", "list", "watch"]
# Full Route management in this namespace only
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["httproutes", "tcproutes", "grpcroutes"]
  verbs: ["*"]
# ReferenceGrant in this namespace (for backend references)
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["referencegrants"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ana-app-developer-prod
  namespace: production
subjects:
- kind: User
  name: ana@company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: gateway-app-developer
  apiGroup: rbac.authorization.k8s.io
---
# Repeat for other namespaces (staging, development, etc.)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: staging
  name: gateway-app-developer
rules:
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gateways"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["httproutes", "tcproutes", "grpcroutes"]
  verbs: ["*"]
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["referencegrants"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ana-app-developer-staging
  namespace: staging
subjects:
- kind: User
  name: ana@company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: gateway-app-developer
  apiGroup: rbac.authorization.k8s.io
```

## Cross-Namespace Security Examples

### 1. Route Binding (Namespace Selection)

```yaml
# Gateway allows routes from specific namespaces
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-system
spec:
  gatewayClassName: traefik
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchExpressions:
          - key: kubernetes.io/metadata.name
            operator: In
            values:
            - production
            - staging
            - development
```

**What this means:**
- Only HTTPRoutes from `production`, `staging`, and `development` namespaces can bind to this Gateway
- Other namespaces are blocked automatically
- Safe because `kubernetes.io/metadata.name` is the actual namespace name

### 2. ReferenceGrant (Cross-Namespace Service Access)

```yaml
# Allow HTTPRoute in 'frontend' namespace to access Services in 'backend' namespace
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-frontend-to-backend
  namespace: backend  # Grant placed in TARGET namespace
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: frontend  # SOURCE namespace
  to:
  - group: ""
    kind: Service  # Allow access to Services in 'backend' namespace
---
# HTTPRoute in 'frontend' namespace can now reference backend services
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: frontend
spec:
  parentRefs:
  - name: shared-gateway
    namespace: gateway-system
  rules:
  - backendRefs:
    - name: api-service
      namespace: backend  # Cross-namespace reference (allowed by ReferenceGrant)
      port: 8080
```

### 3. Team-Based Namespace RBAC

```yaml
# Team A - Can only manage routes in their namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a-prod
  name: team-a-route-manager
rules:
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["httproutes", "referencegrants"]
  verbs: ["*"]
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gateways"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-binding
  namespace: team-a-prod
subjects:
- kind: Group
  name: team-a-developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-a-route-manager
  apiGroup: rbac.authorization.k8s.io
```

## Security Best Practices

### 1. Principle of Least Privilege
```yaml
# ❌ BAD - Too permissive
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# ✅ GOOD - Specific permissions
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["httproutes"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
```

### 2. Namespace Isolation
```yaml
# ✅ Use Role (namespaced) instead of ClusterRole when possible
apiVersion: rbac.authorization.k8s.io/v1
kind: Role  # Not ClusterRole
metadata:
  namespace: production  # Specific namespace
```

### 3. Safe Namespace Selection
```yaml
# ✅ SAFE - Use namespace name
selector:
  matchExpressions:
  - key: kubernetes.io/metadata.name
    operator: In
    values: ["trusted-namespace"]

# ⚠️ RISKY - Custom labels can be modified
selector:
  matchLabels:
    env: production  # Anyone who can label namespaces can bypass this
```

## Verification Commands

### Check User Permissions
```bash
# Check what ana can do with HTTPRoutes in production namespace
kubectl auth can-i create httproutes --namespace production --as ana@company.com

# Check all Gateway API permissions for a user
kubectl auth can-i --list --as ana@company.com --namespace production
```

### Debug RBAC Issues
```bash
# Check if ReferenceGrant exists
kubectl get referencegrants -A

# Check Gateway allowedRoutes configuration
kubectl get gateway shared-gateway -o yaml | grep -A 10 allowedRoutes

# Check effective permissions
kubectl auth can-i '*' httproutes --namespace production --as ana@company.com
```

## Summary

**Key Security Concepts:**
1. **Separation of Concerns** - Different roles manage different resources
2. **Namespace Boundaries** - Cross-namespace access requires explicit handshakes
3. **Least Privilege** - Users get only the minimum permissions needed
4. **Explicit Grants** - Cross-namespace references need ReferenceGrant approval

This RBAC model ensures that:
- Infrastructure providers control the platform
- Cluster operators manage gateways and infrastructure routing
- Application developers focus on their app routing without breaking others
- Cross-namespace access is secure and controlled