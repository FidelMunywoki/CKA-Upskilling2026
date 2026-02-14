## Networking Deep Dive

# Kubernetes Networking – CKA Preparation Notes
[![CKA](https://img.shields.io/badge/CKA-Networking-20%25-blue?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
[![KodeKloud Inspired](https://img.shields.io/badge/Source-KodeKloud-orange?style=for-the-badge)](https://kodekloud.com)

Comprehensive, exam-focused notes covering **~20%** of the CKA exam — **Networking** domain.
Based on KodeKloud-style slides + real CKA patterns (Pod networking, CNI debugging, iptables, DNS, services).

---

## Course Objectives – Networking Focus

- Pre-requisites: Switching, Routing, DNS, Namespaces, Docker Networking
- Pod-to-Pod communication (same node & cross-node – no NAT)
- CNI plugins & configuration (bridge, host-local, Weave, etc.)
- Service networking (ClusterIP, NodePort, LoadBalancer)
- Cluster DNS (CoreDNS records)
- Troubleshooting: namespaces, veth, bridge, iptables, routes

---

## 1. Linux Networking Basics – Must-Know Commands

```bash
# Interfaces & Addresses
ip link show
ip addr show
ip addr add 192.168.1.10/24 dev eth0
ip addr del 192.168.1.10/24 dev eth0

# Routing
ip route show
ip route add 192.168.5.0/24 via 192.168.1.1
ip route add default via 192.168.1.254

# IP Forwarding (critical for gateways & NAT)
cat /proc/sys/net/ipv4/ip_forward          # 0 = off
echo 1 > /proc/sys/net/ipv4/ip_forward     # enable (temporary)
# Permanent: /etc/sysctl.conf → net.ipv4.ip_forward = 1  then  sysctl -p
```

Key Concepts

- Switching → L2 (same subnet) – ARP + MAC learning
- Routing → L3 (different subnets) – needs gateway
- Default Gateway → default via <ip>
- NAT / Masquerade → required for Pods → external world

## 2. Network Namespaces – Heart of Pod Isolation

Every Pod runs in its own network namespace.

Classic Lab Pattern (very common in exam)

```bash
# Create namespaces
ip netns add red
ip netns add blue

# Create veth pair (virtual cable)
ip link add veth-red type veth peer name veth-blue

# Move interfaces to namespaces
ip link set veth-red netns red
ip link set veth-blue netns blue

# Assign IPs
ip -n red  addr add 10.0.0.1/24 dev veth-red
ip -n blue addr add 10.0.0.2/24 dev veth-blue

# Bring up
ip -n red  link set veth-red up
ip -n blue link set veth-blue up
ip -n red  link set lo up
ip -n blue link set lo up

# Test
ip netns exec red ping 10.0.0.2 -c 2
```

> **Note:** In real clusters → kubelet + CNI plugin does this automatically for every Pod.

## 3. Bridge + NAT – Foundation of Many CNI Plugins

```bash
# Create bridge
brctl addbr v-net-0    # or ip link add v-net-0 type bridge
ip link set v-net-0 up
ip addr add 192.168.15.5/24 dev v-net-0

# Attach veth to bridge
ip link set veth-red-br master v-net-0

# NAT (masquerade) for external traffic
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 ! -o v-net-0 -j MASQUERADE
```

> **Important:** Most simple CNI setups (bridge + host-local) follow this pattern.

## 4. Kubernetes Pod Networking Requirements (Memorize!)

- Every Pod gets a unique cluster-wide IP
- Pods on same node communicate directly (no NAT)
- Pods on different nodes communicate directly (no NAT)

→ Requires overlay (Flannel, Weave, VXLAN) or routed BGP (Calico).

## 5. Container Network Interface (CNI)

Kubernetes delegates networking to CNI plugins.
Key locations (check during exam!)

```bash
ps aux | grep kubelet | grep cni          # Look for --network-plugin=cni --cni-*

ls /opt/cni/bin/                          # Plugins: bridge, host-local, loopback, portmap...
ls /etc/cni/net.d/                        # Config: 10-bridge.conflist, 00-cilium.conf, etc.

cat /etc/cni/net.d/10-bridge.conf         # Example config
```

Typical bridge + host-local config

```json
{
  "cniVersion": "1.0.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "routes": [{ "dst": "0.0.0.0/0" }]
  }
}
```

## 6. Popular CNI Plugins (Slides Highlight Weave)

Weave Net (easy install – still relevant)

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Check logs:

```bash
kubectl logs -n kube-system ds/weave-net
```

Other common ones: Flannel, Calico, Cilium (eBPF), Multus (multi-network).

## 7. Cluster DNS – CoreDNS (High-Yield)

Automatic DNS records

- Service: <service>.<namespace>.svc.cluster.local
- Pod (if hostname + subdomain): <pod-ip-dashed>.<namespace>.pod.cluster.local

Quick checks

```bash
# Inside a Pod
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl exec -it <pod> -- nslookup my-svc.default.svc.cluster.local

# CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

## 8. Troubleshooting Quick Reference Table

| Goal | Command / Action |
|---|---|
| Pod IP & node | `kubectl get pod -o wide` |
| Test DNS from Pod | `kubectl exec -it <pod> -- nslookup <service>` |
| CNI config files | `ls /etc/cni/net.d/` and `cat /etc/cni/net.d/*` |
| iptables NAT rules | `iptables -t nat -L -n -v` |
| Node-to-node Pod connectivity | `ping <pod-ip-on-other-node>` from worker node |
| Required ports | `API: 6443`, `Kubelet: 10250`, `NodePort: 30000-32767` |
| Weave logs | `kubectl logs -n kube-system ds/weave-net` |
| Namespace exec (debug Pod network) | `nsenter -t <pod-pid> -n ip a` or `chroot /host ...` |
