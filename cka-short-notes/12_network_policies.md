# Kubernetes Network Policies – CKA Exam Notes

**Official Documentation**  
https://kubernetes.io/docs/concepts/services-networking/network-policies/

These notes are tailored for the **Certified Kubernetes Administrator (CKA)** exam — focused on reading, writing, modifying, and troubleshooting `NetworkPolicy` YAML manifests quickly under exam time constraints.

## 1. Core Concepts – Must Know

- Operates at **Layer 3 / Layer 4** (IP address + port/protocol: TCP, UDP, SCTP)
- **Default behavior**: All ingress and egress traffic to/from pods is **allowed**
- **Allow-only model** — no explicit `deny` rules; anything not matched is denied
- Multiple NetworkPolicies selecting the same pod are **additive** (union / OR logic)
- For a connection to succeed:  
  - **source pod’s egress policy** must allow the traffic **AND**  
  - **destination pod’s ingress policy** must allow the traffic
- Reply / return traffic for allowed connections is **implicitly permitted**
- Traffic **to/from the node** itself (kubelet, host processes) is **always allowed** — even in default-deny scenarios
- Requires a CNI plugin that supports NetworkPolicy (Calico, Cilium, Weave Net with policy mode; **not** plain Flannel)

## 2. Isolation Rules

- A pod becomes **ingress-isolated** if **at least one** NetworkPolicy selects it **and** includes `Ingress` in `policyTypes`
- Same logic applies for **egress-isolated**
- Once isolated in a direction → **only explicitly allowed traffic** is permitted  
  (plus node → pod traffic for ingress direction)

## 3. NetworkPolicy YAML Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
  namespace: prod                   # ← NetworkPolicies are namespaced!
spec:
  podSelector:                      # which pods this policy applies to
    matchLabels:
      app: backend
  policyTypes:                      # MUST list directions you want to restrict
  - Ingress
  - Egress
  ingress:                          # array of ALLOW rules for incoming traffic
  - from: [...]                     # sources — any match allows the rule
    ports: [...]                    # optional — omitted = all ports/protocols
  egress:                           # array of ALLOW rules for outgoing traffic
  - to: [...]                       # destinations
    ports: [...]

## NetworkPolicy Quick Reference

| Field | When Empty / Omitted | Common Exam Mistake |
| :--- | :--- | :--- |
| **`podSelector: {}`** | Applies to **all pods** in the namespace | Forgetting to limit scope |
| **`policyTypes`** | Defaults to `[Ingress]` if ingress rules exist | Forgetting -> direction remains allow-all |
| **`ports: []`** | **All ports** and all protocols allowed | Often intentionally omitted |
| **`from: []` / `to: []`** | No sources/destinations -> effectively deny | — |

---

## 4. High-Yield Exam Patterns (Memorize & Practice These!)

### 4.1 Default Deny All Ingress (Isolate namespace)
```yaml
spec:
  podSelector: {}               # selects all pods
  policyTypes: [Ingress]        # no ingress rules -> deny all inbound

### 4.2 Default Deny All Egress

```yaml
spec:
  podSelector: {}
  policyTypes: [Egress]         # no egress rules
```

### 4.3 Default Deny Everything (both directions)

```yaml
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

### 4.4 Allow All Ingress (after default-deny)

```yaml
spec:
  podSelector: {}
  ingress:
  - {}                          # empty object = allow from anywhere, any port
  policyTypes: [Ingress]
```

### 4.5 Allow only frontend → backend (same namespace)

```yaml
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### 4.6 Allow ingress from specific namespace

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        env: staging
```

### 4.7 Allow egress to external CIDR (e.g. HTTPS outbound)

```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 10.0.0.0/8
  - ipBlock:
      cidr: 203.0.113.0/24
  ports:
  - protocol: TCP
    port: 443
```

### 4.8 Port Range Example (NodePort range)

```yaml
ports:
- protocol: TCP
  port: 30000
  endPort: 32767
```
## 5. Selector Options (used in `from` or `to`)

```yaml
from:                           # or to:
- podSelector:                  # same namespace only
    matchLabels:
      role: frontend
- namespaceSelector:            # entire namespaces
    matchLabels:
      project: team-a
- ipBlock:
    cidr: 172.17.0.0/16
    except: [172.17.1.0/24]
- podSelector: ...              # rare: AND logic with namespaceSelector
  namespaceSelector: ...
```

## 6. Exam Gotchas & Quick Tips

- **Forgetting policyTypes**: That direction stays unrestricted (allow-all).
- **ipBlock behavior**: Can be unexpected if CNI does SNAT/DNAT (very common with Services).
- **Limitations**: Cannot select traffic by Service name, TLS SNI, node name, ICMP type/code.
- **HostNetwork**: Pods running with `hostNetwork: true` usually bypass pod selectors.
- **Loopback**: Pod-to-itself (loopback) traffic cannot be blocked.
- **Existing Connections**: Policy updates do not immediately terminate existing connections.
- **Kubectl help**: Use `kubectl explain networkpolicy.spec` during exam.
- **Strategy**: Apply default-deny → then selectively allow frontend → backend → db.

## 7. Realistic Multi-Policy Example (Frontend → Backend → DB)

```yaml
# Policy 1: frontend → backend:8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# Policy 2: backend → db:5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-backend
spec:
  podSelector:
    matchLabels:
      app: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
---
# Policy 3: Default deny everything else
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

