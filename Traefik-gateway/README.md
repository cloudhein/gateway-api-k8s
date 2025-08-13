# Install Traefik Gateway using Helm

This guide provides step-by-step instructions for installing Traefik Gateway on Kubernetes using Helm.

## Prerequisites

- Kubernetes cluster
- Helm 3.x installed
- kubectl configured to access your cluster

## Installation Steps

### 1. Add Traefik Helm Repository

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

### 2. Download and Install Traefik

```bash
# Pull the specific version
helm pull traefik/traefik --version 37.0.0 --untar

# Install Traefik with custom values
helm install traefik --namespace=traefik --create-namespace traefik/traefik -f values.yaml
```

### 3. Configuration (values.yaml)

```yaml
providers:
  kubernetesGateway:
    enabled: true

ports:
  traefik:
    port: 8080
    expose:
      default: false
  web:
    port: 8000
    expose:
      default: true
  http:
    port: 80
    expose:
      default: true
```

## Verification

### Check Helm Installation

```bash
helm list -A
```

Expected output:
```
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
traefik traefik         1               2025-08-13 08:05:11.725610801 +0700 +07 deployed        traefik-37.0.0  v3.5.0
```

### Check Traefik Resources

```bash
kubectl get all -n traefik
```

Expected output:
```
NAME                           READY   STATUS    RESTARTS   AGE
pod/traefik-76b96769fc-h2wvg   1/1     Running   0          76s

NAME              TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
service/traefik   LoadBalancer   10.132.83.41   172.18.255.180   80:30652/TCP,443:32182/TCP   76s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   1/1     1            1           76s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-76b96769fc   1         1         1       76s
```

### Check Custom Resource Definitions (CRDs)

```bash
kubectl get crds
```

This will show all installed CRDs including Traefik-specific ones like:
- `gatewayclasses.gateway.networking.k8s.io`
- `gateways.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`
- `ingressroutes.traefik.io`
- `middlewares.traefik.io`
- And many more...

## Gateway API Configuration

### 1. Install GatewayClass

First, check if GatewayClass exists:
```bash
kubectl get gatewayclass -A
```

If no resources are found, create the GatewayClass:
```bash
kubectl apply -f gatewayclass.yaml
```

Verify installation:
```bash
kubectl get gatewayclass -A
```

Expected output:
```
NAME      CONTROLLER                      ACCEPTED   AGE
traefik   traefik.io/gateway-controller   True       97m
```

### 2. Install Gateway

```bash
kubectl apply -f gateway.yaml -n traefik
```

Verify Gateway installation:
```bash
kubectl get gateway -A
```

Expected output:
```
NAMESPACE   NAME      CLASS     ADDRESS          PROGRAMMED   AGE
traefik     traefik   traefik   172.18.255.180   True         81m
```

### 3. Install HTTP Route Rules

```bash
kubectl apply -f http-route.yaml
```

Verify HTTP routes:
```bash
kubectl get httproute
```

Expected output:
```
NAME                 HOSTNAMES   AGE
bookinfo-httproute               79m
```

## Configuration Files

You'll need to create the following configuration files:

- `values.yaml` - Helm values configuration (shown above)
- `gatewayclass.yaml` - GatewayClass definition
- `gateway.yaml` - Gateway resource definition
- `http-route.yaml` - HTTPRoute rules

## Troubleshooting

- Ensure your Kubernetes cluster supports Gateway API
- Check that the Traefik service has an external IP assigned
- Verify that all CRDs are properly installed
- Check pod logs: `kubectl logs -n traefik deployment/traefik`

## Advanced Traefik Features

Traefik goes beyond basic routing with powerful middleware capabilities:

### Authentication Middleware

Protect your services with built-in authentication:

```yaml
# BasicAuth middleware
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: traefik
spec:
  basicAuth:
    users:
      - "admin:$2y$10$..."  # htpasswd generated hash
---
# ForwardAuth for OAuth/OIDC
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: oauth-auth
  namespace: traefik
spec:
  forwardAuth:
    address: http://oauth-service:4181
    authResponseHeaders:
      - X-User
      - X-Email
```

### Circuit Breaker

Prevent cascading failures with automatic circuit breaking:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: circuit-breaker
  namespace: traefik
spec:
  circuitBreaker:
    expression: "NetworkErrorRatio() > 0.3 || ResponseCodeRatio(500, 600, 0, 600) > 0.3"
```

### Rate Limiting

Control traffic flow and prevent abuse:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: traefik
spec:
  rateLimit:
    burst: 100
    average: 50
    period: 1m
    sourceCriterion:
      ipStrategy:
        depth: 1
```

### Retry Mechanism

Automatically retry failed requests:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: retry
  namespace: traefik
spec:
  retry:
    attempts: 3
    initialInterval: 100ms
```

### Using Middleware with Gateway API (HTTPRoute)

Apply Traefik middleware to Gateway API HTTPRoutes using ExtensionRef filters:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: secure-app
  namespace: default
spec:
  parentRefs:
  - name: traefik
    namespace: traefik
  hostnames:
  - "secure.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    filters:
    # Traefik middleware via ExtensionRef
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: basic-auth
        namespace: traefik
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: rate-limit
        namespace: traefik
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: circuit-breaker
        namespace: traefik
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: retry
        namespace: traefik
    backendRefs:
    - name: my-app
      port: 80
```

### Using Middleware with IngressRoute (Alternative)

If you prefer simpler middleware syntax, you can also use IngressRoute:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: secure-app-ingress
  namespace: default
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`secure.example.com`)
    kind: Rule
    services:
    - name: my-app
      port: 80
    middlewares:
    - name: basic-auth
      namespace: traefik
    - name: rate-limit
      namespace: traefik
    - name: circuit-breaker
      namespace: traefik
    - name: retry
      namespace: traefik
```

## Next Steps

After successful installation, you can:
1. Configure additional HTTP routes
2. Set up TLS/SSL certificates
3. Configure middleware for authentication, rate limiting, etc.
4. Monitor Traefik through its dashboard (accessible on port 8080)