# ⭐⭐ Scaling & Scheduling — Senior Level

---

## Q1: What is the difference between HPA, VPA, and Cluster Autoscaler?

**Senior Answer:**

These are three different scaling dimensions:

**HPA — Horizontal Pod Autoscaler**
Scales the number of Pod replicas up or down based on metrics (CPU, memory, custom metrics). The workload stays on the same nodes — you're just getting more or fewer copies of the app.
- Example: CPU goes from 30% → 80% → HPA adds more pods (scale out)
- Example: Traffic drops at night → HPA removes pods (scale in)

**VPA — Vertical Pod Autoscaler**
Adjusts the CPU/memory requests and limits of individual Pods based on actual usage. Instead of "more pods," you get "bigger pods."
- Example: Your app consistently uses 400m CPU but you only requested 100m → VPA recommends/sets 450m
- **Warning:** VPA requires a pod restart to apply new resource values. This means it evicts your pod. In production this can cause brief disruption. VPA and HPA should NOT manage the same metric simultaneously — they conflict.

**Cluster Autoscaler**
Adds or removes nodes from the cluster itself. Works at the infrastructure level — calls the cloud provider API (AWS, GCP, Azure) to add EC2/GCE instances.
- When pods are Pending because no node has enough capacity → Cluster Autoscaler provisions a new node
- When nodes are underutilized (< 50%) for 10+ minutes → Cluster Autoscaler drains and terminates the node

**How they work together in production:**
```
Traffic spike:
  HPA adds more pods
    → Pods go Pending (no capacity on existing nodes)
      → Cluster Autoscaler adds a new node
        → Pods schedule on new node ✅

Traffic drops:
  HPA removes pods
    → Nodes become underutilized
      → Cluster Autoscaler removes a node ✅
```

---

## Q2: What are Taints and Tolerations? Give a real use case.

**Senior Answer:**

Taints and tolerations are a mechanism to repel pods from certain nodes — the inverse of node affinity.

A **taint** is applied to a node: "Don't schedule anything here unless you explicitly tolerate this."
A **toleration** is applied to a pod: "I can tolerate nodes with this taint."

**Real use case — GPU nodes:**
GPU instances cost 10x more than regular instances. You don't want regular workloads accidentally scheduling on them and wasting money.

```bash
# Taint all GPU nodes
kubectl taint nodes gpu-node-1 accelerator=nvidia-gpu:NoSchedule

# Only GPU workloads tolerate this taint
```
```yaml
# In your GPU job pod spec:
tolerations:
- key: "accelerator"
  operator: "Equal"
  value: "nvidia-gpu"
  effect: "NoSchedule"
```

**Other common use cases:**
- **Dedicated nodes** for a team: `taint: team=payments:NoSchedule` — only payments team pods run there
- **Spot/preemptible instances**: `taint: cloud.google.com/gke-spot=true:NoSchedule` — only batch jobs that can tolerate interruption run on cheap spot VMs
- **System workloads**: Control plane nodes have `node-role.kubernetes.io/control-plane:NoSchedule` by default — prevents user workloads from running on master nodes

**Taint effects:**
- `NoSchedule` — don't schedule new pods (existing pods stay)
- `NoExecute` — evict existing pods AND don't schedule new ones
- `PreferNoSchedule` — try not to schedule here but it's not a hard rule

---

## Q3: What is Pod Affinity and Anti-Affinity? When would you use each?

**Senior Answer:**

**Node Affinity** — schedule this pod on nodes that match certain labels. It's like `nodeSelector` but with more expressions (In, NotIn, Exists, etc.).

```yaml
# Only schedule on nodes in us-east-1a
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-east-1a"]
```

**Pod Anti-Affinity** — ensure this pod does NOT land on a node that already has certain pods. The most important use case: **high availability.**

```yaml
# Don't put two webapp pods on the same node
# If one node dies, the other pod survives
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: ["webapp"]
      topologyKey: kubernetes.io/hostname
```

**Pod Affinity** — schedule this pod NEAR certain other pods (same node or zone). Use case: a frontend pod that makes hundreds of calls per second to a cache pod — co-locating them eliminates network latency.

**`required` vs `preferred`:**
- `requiredDuringScheduling` — hard rule. Pod stays Pending if no node satisfies it.
- `preferredDuringScheduling` — soft rule. Scheduler tries its best but will ignore it if needed.

In production, I use `preferred` for most affinity rules — `required` can cause pods to stay Pending indefinitely if the cluster is under pressure, which is worse than violating a soft placement preference.

---

## Q4: What is a Resource Quota and a LimitRange?

**Senior Answer:**

**ResourceQuota** — limits total resource consumption for an entire namespace. Prevents one team from consuming all cluster resources.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "10"          # Max 10 CPU cores requested in this namespace
    requests.memory: 20Gi       # Max 20GB memory requested
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"                  # Max 50 pods
    persistentvolumeclaims: "10"
```

When you hit the quota, `kubectl apply` fails with "exceeded quota." Visible in `kubectl describe quota -n team-payments`.

**LimitRange** — sets default and maximum resource requests/limits for individual containers in a namespace. Prevents containers with no resource spec from being unbounded (which means they can starve other workloads).

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:          # If no limits set, use these
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # If no requests set, use these
      cpu: "100m"
      memory: "128Mi"
    max:              # Hard ceiling — cannot exceed these
      cpu: "4"
      memory: "8Gi"
```

**Together:** ResourceQuota controls the namespace total. LimitRange controls per-pod defaults and maximums. In a multi-tenant cluster, you'd apply both to every team namespace.

---

## Q5: How does the Kubernetes scheduler actually decide where to place a pod?

**Senior Answer:**

The scheduler runs two phases:

**Phase 1 — Filtering (finding eligible nodes)**
Eliminate all nodes that cannot run the pod:
- Not enough CPU/memory (requests vs allocatable)
- Node has a taint the pod doesn't tolerate
- Node doesn't match nodeSelector or required affinity
- Node is cordoned (marked unschedulable)
- PVC volume zone mismatch
- Pod requested a specific hostname (nodeName)

If filtering leaves 0 nodes → pod stays Pending.

**Phase 2 — Scoring (picking the best node)**
Among eligible nodes, score each one:
- **LeastRequestedPriority** — prefer nodes with more free resources (spread load)
- **BalancedResourceAllocation** — prefer nodes where CPU and memory utilization are balanced
- **InterPodAffinityPriority** — prefer nodes that satisfy soft affinity/anti-affinity preferences
- **NodeAffinityPriority** — prefer nodes that match preferred node affinity

The node with the highest score wins. Pod is bound to that node (written to etcd). Kubelet on that node picks it up.

**The key insight:** The scheduler only looks at resource **requests**, not actual usage. If you have 50 pods all requesting 100m CPU but actually using 800m, the scheduler thinks the node is lightly loaded. This is why setting accurate resource requests matters — the scheduler is only as smart as the data you give it.
