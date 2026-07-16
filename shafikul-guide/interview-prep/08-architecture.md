# ⭐⭐⭐ Architecture & Design — Principal/Senior Level

---

## Q1: How would you design a Kubernetes setup for a 50-person engineering company?

**Senior Answer:**

I'd think about this across five dimensions:

**1. Cluster topology:**
I wouldn't put everything in one cluster. I'd have at minimum:
- `nonprod` cluster — development + staging, lower cost, spot instances acceptable
- `prod` cluster — production only, on-demand instances, multi-AZ nodes

Separation prevents a bad deployment from affecting prod, limits blast radius of cluster-level mistakes, and makes cost attribution clear.

**2. Namespace strategy:**
In the prod cluster: one namespace per service or team (`payments`, `catalog`, `auth`). Apply ResourceQuotas to each namespace — teams can't starve each other. RBAC per namespace — payments team can't touch catalog.

**3. Node groups:**
- General workload nodes (most services)
- Memory-optimized nodes for caches/DBs
- Spot/preemptible node group for batch jobs and stateless workers (70% cost savings)
- Taint GPU nodes if needed for ML workloads

**4. Platform tooling I'd install:**
- ArgoCD — GitOps deployments
- Cert-manager — automatic TLS certificates
- External Secrets Operator — sync from AWS Secrets Manager
- Prometheus + Grafana + Loki — observability
- Cluster Autoscaler — auto-scale nodes
- NGINX Ingress Controller — HTTP routing
- Falco — runtime security monitoring

**5. Developer experience:**
- Developers get `edit` ClusterRole in their team's namespace on nonprod, `view` on prod
- Standard Helm chart all teams use (`company/service-chart`) — avoids K8s expertise requirement to deploy
- Preview environments: each PR gets an ephemeral namespace (`pr-1234`) spun up by CI

**What I'd NOT do:** Give everyone `cluster-admin`. Use managed K8s (EKS/GKE/AKS) rather than self-managing control plane — the operational cost of managing etcd, upgrades, and control plane HA is not worth it for most teams.

---

## Q2: etcd is the heart of Kubernetes. What happens if it fails? How do you protect it?

**Senior Answer:**

etcd stores all cluster state — every pod, deployment, secret, configmap, everything. If etcd is lost and you have no backup, your cluster configuration is gone. Running pods would continue (kubelet doesn't need etcd to keep containers alive), but you can't make any changes, and the cluster will gradually diverge.

**Protecting etcd in production:**

**1. Run 3 or 5 etcd members (Raft quorum):**
etcd uses the Raft consensus algorithm. With 3 members, the cluster can tolerate 1 failure. With 5 members, it can tolerate 2 failures. Never run 2 or 4 — even numbers have split-brain risk.

**2. Separate etcd nodes:**
In production, etcd should run on dedicated nodes, not mixed with worker workloads. etcd is extremely I/O sensitive — a noisy neighbor doing heavy disk I/O can cause etcd to miss heartbeats and trigger leader elections.

**3. Use fast SSDs:**
etcd performance is directly tied to disk write latency. AWS io2 or GP3, GCP SSD PD. Avoid network-attached spinning disks.

**4. Regular backups:**
```bash
# Snapshot etcd to S3/GCS daily
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://etcd:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```
Test restores quarterly. A backup you've never restored is a backup you can't trust.

**5. For most teams:** Use managed K8s (EKS, GKE, AKS). Google/Amazon manages etcd for you. They run multiple etcd members across availability zones, handle backups, and provide SLAs. The operational complexity of self-managed etcd is enormous.

---

## Q3: What are admission controllers and what can you do with them?

**Senior Answer:**

Admission controllers are plugins that intercept API server requests after authentication and authorization but before the request is persisted to etcd. They're the last line of enforcement before a resource is created.

**Two types:**

**Validating admission webhooks** — can approve or reject a request. "This pod requests `hostNetwork: true` — reject it."

**Mutating admission webhooks** — can modify the request before it's persisted. "Add a sidecar container to every pod in production namespace automatically."

**Real examples of what companies use them for:**

1. **OPA/Gatekeeper policies:**
   - "All deployments must have resource limits" → reject any without them
   - "No image tag `latest` in production" → reject
   - "All containers must have readiness probes" → reject if missing

2. **Istio service mesh injection:**
   - Mutating webhook automatically injects the Envoy sidecar proxy into every pod. Developers don't configure this — it happens transparently.

3. **Security policy:**
   - "Reject any pod running as root (UID 0)"
   - "Reject privileged containers"
   - "Reject containers mounting `/var/run/docker.sock`"

4. **Labeling/annotation enforcement:**
   - "All pods must have `team`, `service`, and `environment` labels" → used for cost attribution and alert routing

**Why this matters in interviews:** Admission controllers are where platform teams implement company-wide policy. If you're building a platform for other teams, this is your enforcement layer — not documentation, not code reviews.

---

## Q4: How would you handle a situation where K8s is the wrong solution?

**Senior Answer:**

This is a judgment question — they want to know if you use K8s thoughtfully or if you apply it to everything because it's fashionable.

**Kubernetes is wrong for:**

**Tiny teams / single apps:**
If you have a 3-person startup with one monolith and one database, Kubernetes adds enormous operational complexity for no benefit. Docker Compose locally + a managed PaaS (Fly.io, Railway, Render, Heroku) or even a simple VM with a systemd service is faster to ship, easier to maintain, and cheaper to run.

**Latency-critical stateful workloads:**
Databases generally shouldn't run on Kubernetes unless your team has deep expertise. StatefulSets have operational quirks (see Lab 09), storage is complex, and K8s node migrations introduce latency. Most companies run their primary databases on managed cloud services (RDS, CloudSQL, PlanetScale) and deploy only stateless services to K8s.

**Simple cron jobs:**
If you just need to run a script every night, a managed cron service (AWS EventBridge + Lambda, GCP Cloud Scheduler + Cloud Run) is vastly simpler than running a Kubernetes CronJob.

**When K8s is clearly worth it:**
- Multiple microservices (5+) with different scaling needs
- Need for zero-downtime deployments with rollback
- Multiple environments (dev/staging/prod) with different configs
- Team > 10 engineers where self-service deployment matters
- Need to run batch workloads alongside serving workloads

**What I'd say to impress:** "I think K8s earns its complexity at around 5-10 microservices or 10+ engineers. Below that, the operational overhead exceeds the benefit. I've seen teams spend 3 months setting up K8s when they should have been building their product."

---

## Q5: What is a Service Mesh and when do you need one?

**Senior Answer:**

A service mesh (Istio, Linkerd, Cilium) is an infrastructure layer that handles service-to-service communication — traffic management, mTLS encryption, observability, and circuit breaking — transparently, without application code changes.

It works by injecting a proxy sidecar (typically Envoy) into every pod. All network traffic goes through the proxy, which applies policies.

**What it provides:**

- **mTLS** — every service-to-service call is encrypted and mutually authenticated. Without this, traffic inside the cluster is plaintext.
- **Traffic management** — canary deployments (send 5% of traffic to v2), circuit breaking (if service B is slow, stop sending requests and return cached responses), retry policies
- **Observability** — automatic distributed tracing (Jaeger/Zipkin), per-service latency/error-rate metrics, without any app code changes
- **Zero-trust networking** — explicit authorization policies: "service A can call service B on endpoint /api only"

**When you DO need it:**
- Compliance requires encrypted in-cluster traffic (PCI-DSS, HIPAA)
- You need canary deployments at the network layer
- You have enough services that distributed tracing and per-service metrics are essential for debugging

**When you DON'T need it:**
- Small number of services
- Team isn't ready for the operational complexity — Istio especially has a steep learning curve and can introduce its own bugs and latency
- Your latency budget is very tight — sidecar proxy adds ~1-2ms per hop

**Honest take:** Istio is powerful and widely used, but I've seen teams adopt it too early and spend months debugging Istio issues instead of building features. Linkerd is simpler if you just need mTLS and basic observability. Cilium does it at the kernel level (no sidecar) with eBPF — the most promising direction for 2025+.
