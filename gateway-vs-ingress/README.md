# Kubernetes Gateway API

A comprehensive guide to Kubernetes Gateway API capabilities and configurations.

## Overview

The Kubernetes Gateway API provides advanced HTTP routing and traffic management capabilities for modern cloud-native applications. It offers a more expressive, extensible, and role-oriented API compared to traditional Ingress controllers.

## Key Features

### ✅ HTTP Header-Based Matching
Route requests based on HTTP headers, allowing for flexible routing decisions.

### ✅ HTTP Method-based Matching
Route traffic based on HTTP methods (GET, POST, PUT, DELETE, etc.)

### ✅ HTTP Request Header Modification  
Modify and manipulate HTTP request headers

### ✅ HTTP Request Redirect
Implement HTTP redirections with custom status codes

### ✅ HTTP Traffic Splitting
Distribute traffic across multiple backend services with weighted routing

### ✅ HTTP Query Parameter-based Matching
Route requests based on query parameters

### ✅ HTTP Response Header Manipulation
Modify response headers before sending to clients

### ✅ HTTP Request Mirroring
Mirror requests to multiple backends for testing and analysis

### ✅ HTTP URL Rewrite
Rewrite URLs and paths before forwarding to backends

### ✅ Multi-protocol Support
Support for TCP, UDP, TLS, and gRPC protocols

## Configuration Examples

### 1. HTTP Header-Based Matching

### User-Agent Strings
- `Mozilla/5.0 (Windows Phone 10.0; Mobile) rv:107.0) Gecko/107.0 Firefox/107.0`
- `Mozilla/5.0 (Windows Phone 10.0; Android 4.2.1; Microsoft; Lumia 950) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.5060.71 Safari/537.36`
- `Mozilla/5.0 (iPhone; CPU iPhone OS 16_5 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/113.0.5672.12 Mobile/15E148 Safari/604.1`
- `Mozilla/5.0 (Android 13; Mobile; LG-M255; rv:113.0) Gecko/113.0 Firefox/113.0`

```yaml
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: httproute-headermatch
  labels:
    httproute: headermatch
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: mygateway
  hostnames:
  - myapp.mydomain.com
  rules:
  - matches:
    - headers:
        type: RegularExpression
        name: User-Agent
        value: ".*(Android|iPhone|Mobile).*"
    backendRefs:
    - name: mobile
      port: 80
```

### 2. HTTP Method-based Matching

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: httproute-methodmatch
  labels:
    httproute: methodmatch
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: my-gateway
  hostnames:
  - myapp.mydomain.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/v2
      method: GET
    backendRefs:
    - name: api-v2
      port: 9001
```

**Example Request:**
```
GET http://myapp.mydomain.com/api/v2/...
```

This configuration will:
- ✅ Match GET requests to `/api/v2` path
- ❌ Block POST, PUT, DELETE and other methods

### 3. HTTP Request Header Modification

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: httproute-requestheadermod
  labels:
    httproute: requestheadermod
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: my-gateway
  hostnames:
  - myapp.mydomain.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /test
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        set:
        - name: Authorization
          value: "Basic dGhldXNlcjp0aGVwYXNzd29yZA=="
    backendRefs:
    - name: api-svc
      port: 80
```

**Example Flow:**
```
Client Request: http://myapp.mydomain.com/test
↓
Gateway adds: Authorization: "Basic dGhldXNlcjp0aGVwYXNzd29yZA=="
↓ 
Backend receives request with injected auth header
```

### 4. HTTP Request Redirect

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: httproute-requestredirect
  labels:
    httproute: requestredirect
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: my-gateway
  hostnames:
  - myapp.mydomain.com
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        hostname: new.myapp.mydomain.com
        statusCode: 302
```

**Example Flow:**
```
Client Request: http://myapp.mydomain.com
↓
Gateway Response: HTTP 302 Redirect
Location: http://new.myapp.mydomain.com
↓
Client follows redirect to new hostname
```

### 5. HTTP Traffic Splitting

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: httproute-trafficsplit
  labels:
    httproute: trafficsplit
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: my-gateway
  hostnames:
  - myapp.mydomain.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /monitoring
    backendRefs:
    - name: m-frontend-v1
      port: 9010
      weight: 95
    - name: m-frontend-v2
      port: 9010
      weight: 5
```

**Traffic Distribution:**
```
http://myapp.mydomain.com/monitoring

95% → m-frontend-v1 (Production)
 5% → m-frontend-v2 (Canary)
```

## Use Cases

### Production Deployments
- **Blue-Green Deployments**: Use traffic splitting to gradually shift traffic between versions
- **Canary Releases**: Route small percentage of traffic to new versions for testing
- **A/B Testing**: Split traffic based on headers or query parameters

### Security & Authentication
- **Header Injection**: Add authentication headers automatically
- **Request Filtering**: Block requests based on methods or headers
- **SSL Termination**: Handle TLS termination at the gateway level

### Development & Testing
- **Request Mirroring**: Copy production traffic to staging environments
- **Path Rewriting**: Map external URLs to internal service paths
- **Header Manipulation**: Add debugging headers or modify responses

## Architecture Benefits

### Role-oriented Design
- **Platform Engineers**: Configure Gateway and infrastructure
- **Application Developers**: Define HTTPRoutes for their services
- **Security Teams**: Implement policies and filters

### Extensibility
- Custom filters and policies
- Support for multiple protocols
- Vendor-neutral specification

### Observability
- Built-in metrics and logging
- Request tracing capabilities
- Traffic monitoring and analysis

## Getting Started

1. **Install a Gateway API Implementation** (e.g., Istio, Contour, or HAProxy Ingress)
2. **Create a Gateway Resource** to define listeners and protocols
3. **Configure HTTPRoutes** to define routing rules
4. **Apply Filters** for advanced traffic management

## Resources

- [Gateway API Official Documentation](https://gateway-api.sigs.k8s.io/)
- [Gateway API Conformance](https://gateway-api.sigs.k8s.io/concepts/conformance/)
- [Implementation Status](https://gateway-api.sigs.k8s.io/implementations/)

---

*This README covers the core capabilities of Kubernetes Gateway API. For implementation-specific features and advanced configurations, consult your Gateway controller's documentation.*