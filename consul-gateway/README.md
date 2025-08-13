# Consul API Gateway Installation with Helm

This guide provides step-by-step instructions to install and configure the Consul API Gateway using Helm in a Kubernetes cluster, set up Gateway API resources, and configure microservices for the Consul service mesh with mTLS and HTTP routing.

## Prerequisites
- Kubernetes cluster (e.g., kind, minikube, or a managed cluster)
- Helm 3 installed
- `kubectl` configured to interact with your cluster
- Basic understanding of Kubernetes, Helm, and Consul service mesh

## Installation Steps

### 1. Add and Update Helm Repository
Add the HashiCorp Helm repository and update it to ensure you have the latest charts.

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### 2. Pull and Install Consul
Pull the Consul Helm chart (version 1.8.0) and install it with the provided `values.yaml` file. The installation creates a `consul` namespace.

```bash
helm pull hashicorp/consul --version 1.8.0 --untar
helm install consul hashicorp/consul --version 1.8.0 --values values.yaml --create-namespace --namespace consul
```

**values.yaml**:
```yaml
global:
  name: consul
connectInject:
  enabled: true
  apiGateway:
    manageExternalCRDs: true
```

### 3. Verify Helm Installation
Check that the Consul Helm release is deployed successfully.

```bash
helm list -A
```

**Expected Output**:
```
NAME    NAMESPACE  REVISION  UPDATED                             STATUS    CHART         APP VERSION
consul  consul     1         2025-08-13 14:54:43.92999386 +0700  deployed  consul-1.8.0  1.21.3
```

### 4. Verify Consul Custom Resource Definitions (CRDs)
Confirm that Consul’s CRDs, including those for the Gateway API, are installed.

```bash
kubectl get crds
```

**Expected Output (partial)**:
```
NAME                                             CREATED AT
gatewayclasses.gateway.networking.k8s.io         2025-08-13T08:40:45Z
gateways.gateway.networking.k8s.io               2025-08-13T08:40:45Z
httproutes.gateway.networking.k8s.io             2025-08-13T08:40:45Z
servicedefaults.consul.hashicorp.com             2025-08-13T08:40:45Z
...
```

### 5. Check Initial Consul Resources
Verify Consul’s pods and services in the `consul` namespace before deploying the Gateway.

```bash
kubectl get all -n consul
```

**Expected Output**:
```
NAME                                               READY   STATUS    RESTARTS   AGE
pod/consul-connect-injector-5fdb7547d-bwtbb        1/1     Running   0          3m9s
pod/consul-server-0                                1/1     Running   0          3m9s
pod/consul-webhook-cert-manager-56d6d7dd49-jmd7s   1/1     Running   0          3m9s

NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                            AGE
service/consul-connect-injector   ClusterIP   10.132.199.248   <none>        443/TCP                                                                            3m9s
service/consul-dns                ClusterIP   10.132.238.100   <none>        53/TCP,53/UDP                                                                      3m9s
service/consul-server             ClusterIP   None             <none>        8500/TCP,8502/TCP,8301/TCP,8301/UDP,8302/TCP,8302/UDP,8300/TCP,8600/TCP,8600/UDP   3m9s
service/consul-ui                 ClusterIP   10.132.102.184   <none>        80/TCP                                                                             3m9s

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/consul-connect-injector       1/1     1            1           3m9s
deployment.apps/consul-webhook-cert-manager   1/1     1            1           3m9s

NAME                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/consul-connect-injector-5fdb7547d        1         1         1       3m9s
replicaset.apps/consul-webhook-cert-manager-56d6d7dd49   1         1         1       3m9s

NAME                             READY   AGE
statefulset.apps/consul-server   1/1     3m9s
```

### 6. Configure Gateway API

#### Verify GatewayClass
The `consul` GatewayClass is automatically installed with the Consul Helm chart.

```bash
kubectl get gatewayclass
```

**Expected Output**:
```
NAME     CONTROLLER                                ACCEPTED   AGE
consul   consul.hashicorp.com/gateway-controller   True       39m
```

#### Deploy Gateway
Apply the Gateway resource to create the Consul API Gateway, which adds a LoadBalancer service for north-south traffic.

```bash
kubectl apply -f gateway.yaml -n consul
```

**gateway.yaml**:
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: consul-gateway
  namespace: consul
spec:
  gatewayClassName: consul
  listeners:
  - protocol: HTTP
    port: 80
    name: http
    allowedRoutes:
      namespaces:
        from: All
```

Verify the Gateway and new resources:

```bash
kubectl get gateway -A
```

**Expected Output**:
```
NAMESPACE   NAME             CLASS    ADDRESS          PROGRAMMED   AGE
consul      consul-gateway   consul   172.18.255.180   True         110s
```

Check updated Consul resources (note the new `consul-gateway` pod and LoadBalancer service):

```bash
kubectl get all -n consul
```

**Expected Output**:
```
NAME                                               READY   STATUS    RESTARTS   AGE
pod/consul-connect-injector-5fdb7547d-bwtbb        1/1     Running   0          21m
pod/consul-gateway-c6558b8cf-5gwwr                 1/1     Running   0          12m
pod/consul-server-0                                1/1     Running   0          21m
pod/consul-webhook-cert-manager-56d6d7dd49-jmd7s   1/1     Running   0          21m

NAME                              TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                                            AGE
service/consul-connect-injector   ClusterIP      10.132.199.248   <none>           443/TCP                                                                            21m
service/consul-dns                ClusterIP      10.132.238.100   <none>           53/TCP,53/UDP                                                                      21m
service/consul-gateway            LoadBalancer   10.132.207.244   172.18.255.180   80:30170/TCP                                                                       12m
service/consul-server             ClusterIP      None             <none>           8500/TCP,8502/TCP,8301/TCP,8301/UDP,8302/TCP,8302/UDP,8300/TCP,8600/TCP,8600/UDP   21m
service/consul-ui                 ClusterIP      10.132.102.184   <none>           80/TCP                                                                             21m

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/consul-connect-injector       1/1     1            1           21m
deployment.apps/consul-gateway                1/1     1            1           12m
deployment.apps/consul-webhook-cert-manager   1/1     1            1           21m

NAME                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/consul-connect-injector-5fdb7547d        1         1         1       21m
replicaset.apps/consul-gateway-c6558b8cf                 1         1         1       12m
replicaset.apps/consul-webhook-cert-manager-56d6d7dd49   1         1         1       21m

NAME                             READY   AGE
statefulset.apps/consul-server   1/1     21m
```

**Note**: Applying `gateway.yaml` creates the `consul-gateway` pod and a LoadBalancer service (`service/consul-gateway`) with an external IP (`172.18.255.180`) for north-south traffic.

#### Deploy HTTPRoute
Apply the HTTPRoute resource to define routing rules for your microservices.

```bash
kubectl apply -f http-route.yaml
```

**http-route.yaml**:
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: bookinfo-http-route
  namespace: default
spec:
  hostnames:
  - "productpage.local"

  parentRefs:
  - name: consul-gateway # Reference to the Gateway
    namespace: consul

  rules:
  - matches:
    - path: 
        type: PathPrefix
        value: /

    backendRefs:
    - kind: Service
      name: productpage
      port: 9080
      #namespace: default
```

Verify the HTTPRoute:

```bash
kubectl get httproute
```

**Expected Output**:
```
NAME                  HOSTNAMES               AGE
bookinfo-http-route   ["productpage.local"]   5m42s
```

### 7. Enable Consul Service Mesh for Microservices
Patch your microservice deployments to enable Consul’s service mesh by injecting Envoy sidecar proxies for service discovery and mTLS.

```bash
kubectl patch deployment productpage-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"consul.hashicorp.com/connect-inject":"true"}}}}}'
kubectl patch deployment details-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"consul.hashicorp.com/connect-inject":"true"}}}}}'
kubectl patch deployment reviews-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"consul.hashicorp.com/connect-inject":"true"}}}}}'
kubectl patch deployment reviews-v2 -p '{"spec":{"template":{"metadata":{"annotations":{"consul.hashicorp.com/connect-inject":"true"}}}}}'
kubectl patch deployment reviews-v3 -p '{"spec":{"template":{"metadata":{"annotations":{"consul.hashicorp.com/connect-inject":"true"}}}}}'
kubectl patch deployment ratings-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"consul.hashicorp.com/connect-inject":"true"}}}}}'
```

### 8. Configure ServiceDefaults for HTTP Protocol
Apply `ServiceDefaults` to configure the HTTP protocol for your services (Consul defaults to TCP without this).

```bash
kubectl apply -f servicedefault.yaml
```

**servicedefault.yaml**:
```yaml
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceDefaults
metadata:
  name: productpage
  namespace: default
spec:
  protocol: "http"
---
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceDefaults
metadata:
  name: details
  namespace: default
spec:
  protocol: "http"
---
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceDefaults
metadata:
  name: reviews
  namespace: default
spec:
  protocol: "http"
---
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceDefaults
metadata:
  name: ratings
  namespace: default
spec:
  protocol: "http"
```

Verify ServiceDefaults:

```bash
kubectl get servicedefaults
```

**Expected Output**:
```
NAME          SYNCED   LAST SYNCED   AGE
details       True     12s           12s
productpage   True     12s           12s
ratings       True     11s           12s
reviews       True     11s           12s
```

### 9. Test the Gateway API
Test access to your application via the Consul Gateway’s external IP.

```bash
curl -H "Host: productpage.local" http://172.18.255.180
```

**Note**: If the `curl` command fails (e.g., `Couldn't connect to server`), ensure:
- The `consul-gateway` service’s external IP (`172.18.255.180`) is accessible.
- Your microservices are running and healthy.
- The `ServiceDefaults` and sidecar injections are correctly applied.
- Your network allows traffic to the Gateway’s port (80).

### Troubleshooting
- **Check Pod Logs**:
  ```bash
  kubectl logs <pod-name> -n default -c consul-dataplane
  ```
- **Verify Service Registration**:
  Use the Consul CLI or UI to confirm services are registered:
  ```bash
  consul catalog services -namespace=default
  ```
- **Inspect Gateway Status**:
  ```bash
  kubectl describe gateway consul-gateway -n consul
  ```
- **Check Sidecar Injection**:
  Ensure the `consul-dataplane` container is running in your pods:
  ```bash
  kubectl describe pod <pod-name> -n default
  ```

## Notes
- The Consul service mesh automatically enables mTLS for secure service-to-service communication.
- The `consul-gateway` LoadBalancer service provides an external IP for north-south traffic, created after applying `gateway.yaml`.
- Ensure your `values.yaml` enables sidecar injection and Gateway API support.

## License
This project is licensed under the MIT License.