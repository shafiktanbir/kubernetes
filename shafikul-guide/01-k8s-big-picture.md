# 01 — Kubernetes Big Picture
> Read this before touching any code. It gives you the mental model.
> After this doc you should be able to hold a conversation in any SIG meeting.

---

## What Problem Kubernetes Solves

You have an application. You want it to:
- Run reliably across many machines
- Restart automatically if it crashes
- Scale up when traffic spikes, down when it drops
- Roll out new versions without downtime
- Be configured the same way in dev, staging, and prod

Before Kubernetes, you solved each of these problems separately with shell scripts, custom tooling, and prayer. Kubernetes solves all of them through one unified API.

---

## The Core Idea: Desired State vs Actual State

This is the single most important concept. Internalize it before anything else.

```
You declare: "I want 3 replicas of my web server running"
Kubernetes observes: "There are currently 2 running"
Kubernetes acts: "I'll create 1 more"
Kubernetes observes: "There are now 3 running"
Kubernetes rests: "Nothing to do"
```

This is called **reconciliation**. Every single component in Kubernetes does this loop:

```
loop forever:
    observe actual state
    compare to desired state
    take the smallest action to close the gap
    wait for next event
```

You never tell Kubernetes "create a pod". You tell it "the desired state is: a pod exists". Kubernetes figures out the difference and acts. This is called **declarative configuration**.

---

## Kubernetes Is NOT a Container Runner

Common misconception: Kubernetes runs containers.

**Truth:** Kubernetes *manages* container runners. It tells `containerd` or `CRI-O` what to do. Kubernetes itself never touches a container. The `kubelet` sends gRPC calls to the container runtime interface (CRI). The runtime calls `runc` to create Linux namespaces. Kubernetes is the brain; the container runtime is the hands.

```
Kubernetes (brain)
    ↓ gRPC (CRI)
containerd / CRI-O (runtime)
    ↓ OCI spec
runc (actually creates namespaces/cgroups)
    ↓
Your container
```

---

## Core Concepts — The Vocabulary

You must know these before any SIG conversation.

### Pod
The smallest deployable unit. One or more tightly coupled containers that share:
- Network namespace (same IP, same ports)
- Storage volumes
- Lifecycle (live and die together)

A Pod is **not** long-lived. Think of it as ephemeral. Controllers create and delete Pods.

### Node
A physical or virtual machine running a kubelet. Nodes are the workers.

```
Control Plane Node          Worker Nodes
┌──────────────────┐   ┌──────────┐  ┌──────────┐
│ kube-apiserver   │   │ kubelet  │  │ kubelet  │
│ kube-scheduler   │   │ kube-    │  │ kube-    │
│ controller-mgr   │   │ proxy    │  │ proxy    │
│ etcd             │   │          │  │          │
└──────────────────┘   └──────────┘  └──────────┘
```

### Namespace
A virtual cluster within a cluster. Provides name scoping and resource quotas. Most objects are namespaced (`Pods`, `Services`, `Deployments`). Some are cluster-scoped (`Nodes`, `PersistentVolumes`, `ClusterRoles`).

### Workload Resources (Controllers that manage Pods)

| Resource | What it does |
|----------|-------------|
| `Deployment` | Manages a ReplicaSet. Handles rolling updates. |
| `ReplicaSet` | Ensures N copies of a Pod spec are running. |
| `StatefulSet` | Like Deployment but with stable network identity and ordered rollout. |
| `DaemonSet` | Runs one Pod per node (e.g., log collectors, monitoring agents). |
| `Job` | Runs Pod(s) to completion. |
| `CronJob` | Runs a Job on a schedule. |

### Service
A stable virtual IP (ClusterIP) that load-balances to a set of Pods selected by labels. Pods come and go; the Service IP stays constant. `kube-proxy` implements Services using iptables/IPVS rules.

### ConfigMap / Secret
Configuration data injected into Pods as environment variables or volume files. `Secret` is base64-encoded (not encrypted by default — encryption at rest is a separate feature).

### PersistentVolume (PV) / PersistentVolumeClaim (PVC)
Persistent storage. A `PV` is a piece of storage (disk, NFS, cloud volume). A `PVC` is a Pod's request for storage. The control plane binds PVCs to PVs.

### Label + Selector
Labels are key-value pairs attached to any object. Selectors query labels to find objects. This is how Services find Pods, how ReplicaSets own Pods, how NetworkPolicies target Pods.

```yaml
# Pod has a label
metadata:
  labels:
    app: web
    version: v2

# Service selects Pods with that label
selector:
  app: web
```

### Annotation
Also key-value metadata, but for tooling/humans, not for selectors. Larger, unstructured data goes here (e.g., last-applied-configuration, prometheus scrape config).

### Finalizer
A string in `metadata.finalizers`. Kubernetes will not delete an object until all finalizers are removed. Controllers use finalizers to do cleanup work before deletion.

### Owner Reference
A link from a child object to its owner. ReplicaSets are owned by Deployments. Pods are owned by ReplicaSets. The Garbage Collector deletes children when parents are deleted.

---

## The Control Plane

The control plane is the brain. It does not run your workloads.

```
┌─────────────────────────── Control Plane ──────────────────────────┐
│                                                                      │
│  kubectl ──► kube-apiserver ──► etcd                               │
│                    ▲                                                 │
│                    │                                                 │
│  kube-scheduler ───┘  (reads unscheduled pods, writes nodeName)    │
│  controller-manager ──┘ (reads desired state, writes actual state) │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### kube-apiserver
- The only door into the cluster
- REST API server
- Validates all requests (auth → authz → admission → validation)
- Persists to etcd
- Sends watch events to all subscribers

### etcd
- Distributed key-value store
- The only persistent state in the cluster
- Only kube-apiserver writes to it
- Data is replicated across 3-5 etcd instances for HA

### kube-scheduler
**Job: PLACEMENT only — decides which node a Pod runs on.**

- Watches for Pod objects with no `spec.nodeName` set
- Runs filter + score plugins to pick the best node
- Writes `spec.nodeName` back to API server — then its job is done
- **Does NOT create pods**
- **Does NOT monitor pods**
- **Does NOT restart pods if they crash**
- Once a pod is placed, the scheduler forgets about it

### kube-controller-manager
**Job: MONITORING + MAINTAINING — creates pods and keeps desired count.**

- Watches desired state vs actual state — constantly, forever
- If a pod crashes → creates a new Pod object to replace it
- If you scale from 3 → 5 replicas → creates 2 new Pod objects
- If you scale from 5 → 2 replicas → deletes 3 Pod objects
- **Does NOT decide which node a pod runs on** (that's the scheduler's job)

**The exact division of responsibility:**

| Component | Creates pods? | Monitors pods? | Places pods on a node? | Runs containers? |
|-----------|:---:|:---:|:---:|:---:|
| kube-controller-manager | ✅ Yes | ✅ Yes | ❌ No | ❌ No |
| kube-scheduler | ❌ No | ❌ No | ✅ Yes | ❌ No |
| kubelet | ❌ No | ✅ Yes (on its node) | ❌ No | ✅ Yes |

**The order they work in:**
```
1. controller-manager  →  creates Pod object ("a pod needs to exist")
2. kube-scheduler      →  places it on a node ("put it on node-3")
3. kubelet             →  runs the container ("starting nginx...")
```

If a pod crashes, step 1 repeats. The scheduler and kubelet don't know or care about the crash — only the controller-manager does.

---

## The Data Plane (Worker Nodes)

The data plane actually runs your workloads.

### kubelet
- Runs on every worker node
- Watches the API server for Pods assigned to this node
- Calls the container runtime (via CRI gRPC) to start/stop containers
- Reports node and pod status back to the API server
- Runs probes (liveness, readiness, startup)
- Manages volumes (mounts/unmounts)

### kube-proxy
- Runs on every node
- Programs iptables/IPVS rules for Service routing
- When a Pod connects to `ClusterIP:Port`, kube-proxy's rules redirect it to a real Pod

### Container Runtime
- containerd or CRI-O (not Docker — Docker support was removed in 1.24)
- Receives gRPC calls from kubelet via CRI
- Calls runc to create containers
- Manages image pulling and storage

---

## How State Flows Through the System

> **Important:** The diagram below shows what happens when you create a **bare Pod** directly.
> In real production, nobody creates bare Pods — you create Deployments.
> The kube-controller-manager is essential for the real flow. See below.

### Flow 1: Bare Pod (direct `kubectl apply -f pod.yaml`)

```
kubectl apply -f pod.yaml
    │
    ▼
kube-apiserver (validates + stores Pod)
    │
    ├──► etcd (durable storage)
    │
    ├──► kube-scheduler watches → picks node → writes pod.spec.nodeName
    │
    └──► kubelet on chosen node → calls CRI → container running
              │
              └──► kubelet updates pod.status → kube-apiserver → etcd
```

kube-controller-manager: **NOT involved** here. You created a Pod directly.
If this pod dies, **nothing brings it back**. It's gone.

---

### Flow 2: Deployment (real production usage)

```
kubectl apply -f deployment.yaml   (replicas: 3)
    │
    ▼
kube-apiserver stores Deployment object
    │
    ▼ kube-controller-manager: Deployment controller watches
    │   "desired=3 replicas, actual=0 ReplicaSets" → creates ReplicaSet
    │
    ▼ kube-controller-manager: ReplicaSet controller watches
    │   "desired=3 Pods, actual=0" → creates 3 Pod objects
    │
    ▼ kube-scheduler watches
    │   assigns each Pod to a node
    │
    ▼ kubelet on each node
        starts containers

    Later: one pod crashes / node dies
    │
    ▼ kube-controller-manager: ReplicaSet controller notices
      "desired=3, actual=2" → creates 1 new Pod → scheduler places it → kubelet starts it
```

**This is why kube-controller-manager exists.** It bridges the gap between
what the user declares (Deployment) and what can actually run (Pod).

---

### What the controller-manager does that nothing else can

| Situation | Who handles it |
|-----------|---------------|
| Pod dies inside a Deployment | ReplicaSet controller recreates it |
| Node goes offline — pods evicted | Node lifecycle controller → ReplicaSet controller recreates elsewhere |
| You update the image in a Deployment | Deployment controller orchestrates the rolling update |
| You delete a Deployment | Garbage Collector controller deletes its ReplicaSets and Pods |
| Job needs to run to completion | Job controller tracks completions and retries failures |
| CronJob needs to fire at 2am | CronJob controller creates a Job at the scheduled time |

**The rule:** Scheduler and kubelet only know about **Pods**. They don't know what a Deployment is.
The controller-manager is the layer that translates everything else → Pods.

Nothing is pushed. Everything is pulled via watches. Every component is independently reconciling.

---

## API Machinery — How the API Is Structured

Every Kubernetes object has:

```yaml
apiVersion: apps/v1         # Group/Version
kind: Deployment             # Resource type
metadata:
  name: my-app
  namespace: default
  labels: {}
  annotations: {}
  resourceVersion: "12345"  # Optimistic concurrency token
  uid: abc-123              # Immutable, unique identifier
spec:                        # Desired state (you write this)
  replicas: 3
status:                      # Actual state (Kubernetes writes this)
  readyReplicas: 3
```

**The API is versioned:** `v1`, `v1beta1`, `v1alpha1`. Alpha features can be removed. Beta are stable but may change. GA (`v1`) will not break.

**API groups organize resources:**
- `core` (no group prefix) — Pod, Node, Service, ConfigMap, Secret
- `apps` — Deployment, ReplicaSet, StatefulSet, DaemonSet
- `batch` — Job, CronJob
- `rbac.authorization.k8s.io` — ClusterRole, RoleBinding
- `networking.k8s.io` — NetworkPolicy, Ingress
- `storage.k8s.io` — StorageClass, CSIDriver

---

## The Release Cycle

Kubernetes releases three times per year (roughly every 4 months). Each release is versioned: `v1.30`, `v1.31`, `v1.32`.

**Enhancement lifecycle:**
1. Proposal (KEP — Kubernetes Enhancement Proposal) written in `kubernetes/enhancements` repo
2. Feature implemented behind a feature gate (alpha, off by default)
3. Enabled by default in beta
4. Graduates to stable (GA) — feature gate deprecated and eventually removed

**Finding KEPs:** https://github.com/kubernetes/enhancements/tree/master/keps

If you want to understand why code exists the way it does, find the KEP that introduced it.

---

## Key Terms You Will Hear in SIG Meetings

| Term | Meaning |
|------|---------|
| **KEP** | Kubernetes Enhancement Proposal — design doc for a feature |
| **LGTM** | "Looks Good To Me" — reviewer approval |
| **Approve** | Final sign-off from a directory owner (OWNERS file) |
| **Triage** | Classifying and routing issues |
| **Flake / Flaky test** | A test that sometimes passes and sometimes fails without code changes |
| **API machinery** | The generic plumbing that handles any API object (serialization, versioning, storage) |
| **In-tree** | Code inside the kubernetes/kubernetes repo |
| **Out-of-tree** | Code in a separate repo (e.g., a CSI driver, a scheduler plugin) |
| **CRD** | Custom Resource Definition — extending the API with your own types |
| **Operator** | A controller + CRD that manages a stateful application |
| **Control loop** | The reconcile loop: observe → diff → act |
| **Informer** | A cached watch on an API resource — the foundation of all controllers |
| **Lister** | Read-only access to an informer's local cache |
| **WorkQueue** | A rate-limited queue controllers use to process events |
| **Feature gate** | A boolean flag that enables/disables a feature at runtime |
| **Admission** | Policy enforcement at API request time (before storage) |
| **GVK** | Group / Version / Kind — fully qualified type identifier |
| **GVR** | Group / Version / Resource — REST endpoint identifier |
| **Owner reference** | A link from a child object to its owning parent |
| **Finalizer** | A deletion gate — object won't be deleted until all finalizers are cleared |
| **resourceVersion** | Optimistic concurrency token — used for conflict detection |
| **Watch** | A long-lived HTTP/2 connection that streams object change events |
| **Backoff** | Retry with exponential delay — used in controllers to avoid thundering herd |
| **SLO/SLI** | Service Level Objective/Indicator — performance targets (e.g., API latency) |

---

## What "Cloud Native" Actually Means

Cloud native = designed to run on ephemeral, distributed infrastructure.

The implications for Kubernetes code:
- **No local state.** If a component restarts, it must reconstruct its state from the API server.
- **Idempotent.** Every operation must be safe to retry. `Create` should not fail if the object already exists — check with `IsAlreadyExists()` and proceed.
- **Level-triggered, not edge-triggered.** Controllers don't react to "the pod was deleted" — they react to "the desired count is 3, actual is 2". This makes them resilient to missed events.
- **Eventual consistency.** State takes time to propagate. A Pod might be "Running" in etcd while kubelet hasn't started it yet. Design for this lag.
