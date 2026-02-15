# Ingress & Gateway API – CKA Exam Notes

Detailed notes focused on **Ingress** and **Gateway API** for the Certified Kubernetes Administrator (CKA) exam.  
Covers key concepts, YAML examples, commands, common exam tasks (create/configure, troubleshoot, migration), and differences.

**Note (2025+ curriculum)**: Ingress remains core and widely used. Gateway API is increasingly tested — especially migration tasks (Ingress → Gateway API) and basic Gateway + HTTPRoute creation.

## 1. Ingress in Kubernetes

### Purpose
Manages external **HTTP/HTTPS** access to Services using Layer 7 routing (host-based, path-based).  
Provides name-based virtual hosting and path routing with **one external LoadBalancer/IP**.

### Why not just use Services?
- `ClusterIP` → internal only
- `NodePort` → high ports on all nodes (not production-friendly for HTTP)
- `LoadBalancer` → one external LB per Service (expensive, no host/path routing)

Ingress → **one LB + routing rules**.

### Key Components
- **Ingress** resource (`networking.k8s.io/v1`) — declarative rules
- **Ingress Controller** — actual implementation (NGINX, Traefik, HAProxy, Contour, Ambassador, etc.)
- **IngressClass** — selects which controller handles the Ingress

### Basic Ingress YAML Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2   # common rewrite example
spec:
  ingressClassName: nginx     # recommended way (v1.18+)
  # alternative (older): annotations: kubernetes.io/ingress.class: "nginx"
  tls:
  - hosts:
    - foo.bar.com
    secretName: tls-secret    # must exist: tls.crt + tls.key
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /app1(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: bar.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 8080
```

### Path Types (Exam-critical)

| Type | Behavior |
|---|---|
| **Prefix** | Matches if path starts with value (`/app` → `/app`, `/app/`, `/apple`) |
| **Exact** | Must match exactly |
| **ImplementationSpecific** | Controller decides (often behaves like Prefix) |

### TLS Termination

Defined in `spec.tls`.  
Secret must contain `tls.crt` and `tls.key` in the same namespace.

### Common NGINX Annotations (controller-specific)

```
nginx.ingress.kubernetes.io/rewrite-target
nginx.ingress.kubernetes.io/ssl-redirect: "true"
nginx.ingress.kubernetes.io/proxy-body-size: 8m
```

### Useful Commands

```bash
# Install NGINX Ingress Controller (if not already present)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.XX.X/deploy/static/provider/cloud/deploy.yaml

kubectl get ingress
kubectl describe ingress example-ingress
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx <ingress-controller-pod>
```

### Exam Tips – Ingress

- Create Ingress for multi-host / multi-path routing

















- Create Ingress for multi-host / multi-path routing
- Add TLS using existing Secret
- Use `kubectl edit ingress` for quick changes
- Troubleshoot: check controller logs, IngressClass match, Service/Endpoints readiness

## 2. Gateway API (Successor to Ingress)

### Status in CKA

- Added / emphasized in 2025+ curriculum updates
- Ingress is frozen (no new features)
- Gateway API = future direction → expect migration tasks and basic Gateway + HTTPRoute creation

### Why Gateway API > Ingress?

- More extensible and role-oriented (infra vs app separation)
- Native support for TCP/UDP/gRPC (not just HTTP/HTTPS)
- No controller-specific annotations → more portable
- Built-in: traffic splitting, weighted routing, header matching, etc.
- CRDs: GatewayClass, Gateway, HTTPRoute, TCPRoute, ...

### Core Resources

- **GatewayClass** — cluster-wide, selects controller (like IngressClass)
- **Gateway** — defines listeners (ports, protocols, hostname), binds to class
- **HTTPRoute** (or TCPRoute, etc.) — routing rules, references Services

### Simple Gateway + HTTPRoute Example

```yaml
# Gateway (infrastructure / platform team)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-web-gateway
spec:
  gatewayClassName: nginx   # or istio, traefik, cilium, etc.
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All           # or Same / Selector
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: tls-secret

# HTTPRoute (application team)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
  - name: prod-web-gateway
  hostnames:
  - "foo.bar.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app1
    backendRefs:
    - name: app1-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app2-service
      port: 8080
```

### Migration: Ingress → Gateway API (frequent exam task)

1. Create Gateway with appropriate listeners (HTTP/HTTPS + TLS if needed)
2. Create one or more HTTPRoute(s) replicating hosts/paths
3. Reference the same backend Services
4. Test traffic → delete old Ingress resource

### Exam Tips – Gateway API

- CRDs usually pre-installed in recent exam environments
- Use `kubectl explain gateway`, `kubectl explain httproute`
- Common task: expose Service on `/` via Gateway + HTTPRoute
- Troubleshoot: `kubectl describe httproute`, check Gateway status.conditions


| Feature | Ingress | Gateway API |
|---|---|---|
| **API Stability** | Stable (frozen) | Evolving (v1 in progress) |
| **Protocols** | HTTP/HTTPS only | HTTP, HTTPS, TCP, UDP, gRPC, ... |
| **Configuration Style** | Heavy annotations | Structured CRDs, no vendor lock-in |
| **Role Separation** | Limited | Clear (GatewayClass vs Routes) |
| **Advanced Routing** | Limited (controller-dependent) | Built-in: split, weights, headers, etc. |
| **CKA Exam Focus** | Create, configure, TLS, troubleshoot | Migration, basic Gateway + Route |
## Practice Recommendations

Deploy both in Minikube / kind / killer.sh labs
Migrate a working Ingress to Gateway API
Intentionally break things → troubleshoot 404 / no-route / TLS errors


