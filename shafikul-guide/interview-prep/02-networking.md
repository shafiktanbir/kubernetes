# ⭐ Networking — Services, Ingress, DNS, CNI

---

## Q1: How does DNS work inside a Kubernetes cluster?

**Senior Answer:**

K8s runs a DNS server called CoreDNS inside the cluster (as a Deployment in `kube-system`). Every pod has its `/etc/resolv.conf` configured to point to CoreDNS.

When pod A wants to reach service B:
```
1. App code: connect to "db-service"
2. OS resolves: sends DNS query to CoreDNS (10.96.0.10 typically)
3. CoreDNS looks up "db-service" in its K8s records
4. Returns the Service's ClusterIP (e.g., 10.43.0.45)
5. App connects to that IP
6. kube-proxy routes the connection to one of the backing pods
```

**The full DNS naming convention:**
```
<service>.<namespace>.svc.cluster.local
```
Examples:
- `db-service.production.svc.cluster.local` — full FQDN
- `db-service.production` — works from any namespace
- `db-service` — only works within the SAME namespace (search domain is applied)

**Why this matters in interviews:**
Cross-namespace calls need the namespace in the hostname. A common bug is a pod in `frontend` namespace trying to connect to `db-service` (without namespace) — it resolves to nothing, but `db-service.backend` works fine.

**Headless Services (no ClusterIP):**
If you set `clusterIP: None`, CoreDNS returns the individual pod IPs instead of a virtual IP. Used for StatefulSets where clients need to connect to a specific pod by name (`db-0.db-service`, `db-1.db-service`).

---

## Q2: What is a CNI plugin and which ones have you worked with?

**Senior Answer:**

CNI stands for Container Network Interface — the plugin responsible for:
1. Assigning IP addresses to pods
2. Setting up networking so pods on different nodes can communicate
3. Optionally enforcing NetworkPolicy

K8s itself has no built-in networking — it relies entirely on CNI plugins. This is why you must install a CNI before pods can start.

**Common CNI plugins:**

| Plugin | Strengths | Weaknesses |
|--------|-----------|------------|
| **Flannel** | Simple, easy to set up | No NetworkPolicy support |
| **Calico** | NetworkPolicy support, BGP routing | More complex |
| **Cilium** | eBPF-based, best performance, L7 policy | Newest, needs newer kernel |
| **Weave** | Simple, NetworkPolicy support | Being deprecated |

**k3d uses Flannel by default.** This is why NetworkPolicy objects are created but silently ignored in k3d unless you replace Flannel with Calico or Cilium.

**The insight that impresses:** "CNI plugins implement networking differently — Flannel uses VXLAN overlay, Calico can use BGP to route pod traffic without encapsulation overhead (better performance), and Cilium uses eBPF to process networking at the kernel level, bypassing iptables entirely. For high-throughput workloads, CNI choice matters for latency and CPU overhead."

---

## Q3: What is the difference between ClusterIP, NodePort, LoadBalancer, and ExternalName?

**Senior Answer:**

These are the four Service types, operating at different layers:

**ClusterIP (default):**
- Virtual IP only reachable inside the cluster
- Pod-to-pod communication
- Never exposed externally
- Example: frontend pods → backend service → backend pods

**NodePort:**
- Exposes the service on a static port (30000-32767) on every node's external IP
- `http://<any-node-ip>:31234` reaches your service
- Rarely used in production — not load balanced properly, requires knowing node IPs, port range is awkward
- Useful for: bare-metal clusters without cloud LB, quick external testing

**LoadBalancer:**
- Provisions a cloud load balancer (AWS ELB, GCP LB, Azure LB) automatically
- Gives you a stable external IP/DNS
- K8s calls the cloud provider API — requires running on a cloud or with MetalLB on bare-metal
- This is how you expose services externally in production
- Creates a NodePort under the hood — `LoadBalancer → NodePort → ClusterIP → Pods`

**ExternalName:**
- Maps a service name to an external DNS name
- No proxying, no routing — just DNS CNAME
- Example: map `database` to `mydb.us-east-1.rds.amazonaws.com`
- Lets pods use K8s DNS to refer to external services — useful for gradual migration from external to internal

**In production reality:** You don't expose every service via LoadBalancer — each one costs money and gets its own IP. Instead: one LoadBalancer → Ingress controller → all HTTP services routed by hostname/path. This is the standard pattern.

---

## Q4: How does kube-proxy work and what does it actually do?

**Senior Answer:**

kube-proxy runs as a DaemonSet on every node. Its job: maintain network rules that implement the Service abstraction — when a request hits a ClusterIP, kube-proxy routes it to one of the healthy backing pods.

**iptables mode (default):**
kube-proxy watches the API server for Service and Endpoint changes, then programs `iptables` rules on the node. When a packet is destined for a ClusterIP (e.g., 10.43.0.45:80), iptables intercepts it and DNAT's it to a random pod IP (e.g., 10.0.2.5:80) using probability rules.

The problem: iptables rules are O(n) — as the number of services/endpoints grows, the rule table gets enormous and expensive to scan. At 10,000 services, iptables becomes a performance bottleneck.

**ipvs mode (better for large clusters):**
Uses the kernel's IPVS (IP Virtual Server) instead of iptables. O(1) lookup using hash tables. Supports proper load balancing algorithms (round-robin, least-connection, etc.). Recommended for clusters with thousands of services.

**eBPF mode (Cilium, future):**
Cilium replaces kube-proxy entirely with eBPF programs in the kernel. Faster than both iptables and IPVS. No userspace component needed. This is the direction K8s networking is heading.

**Practical implication:** For most clusters under 500 services, iptables mode is fine. At scale, switch to ipvs. If you're designing a high-performance platform, evaluate Cilium with kube-proxy replacement.

---

## Q5: What happens at the network level during a pod-to-pod call across nodes?

**Senior Answer:**

This depends on the CNI, but with a common overlay network (Flannel VXLAN):

```
Pod A (node 1, IP: 10.0.1.5) calls Pod B (node 2, IP: 10.0.2.8)

1. Pod A's container sends packet to 10.0.2.8
2. Packet hits node 1's routing table
   → 10.0.2.0/24 is on a different node
3. Flannel on node 1 encapsulates the packet in a VXLAN UDP packet
   Outer packet: node1-IP → node2-IP
   Inner packet: 10.0.1.5 → 10.0.2.8
4. UDP packet traverses the real network
5. Flannel on node 2 decapsulates the packet
6. Delivers inner packet to Pod B via the virtual bridge (cni0/flannel.1)
```

With **Calico in BGP mode**: no encapsulation — pod IPs are real routes in your network's BGP routing table. The packet from node 1 goes directly via BGP routing. Lower overhead, better performance.

With **Cilium eBPF**: packet is processed in the kernel before it ever reaches userspace. eBPF programs do the routing directly. Fastest option available.

**The key insight:** All pods get real IP addresses that are routable within the cluster. K8s doesn't use NAT for pod-to-pod traffic (only for outbound internet traffic). This is why you can always connect directly to a pod IP for debugging — it's a real routable address within the cluster network.
