# 03 — Request Flow: kubectl apply -f pod.yaml
> The most useful thing to understand. Every important concept appears in this flow.
> Trace this once with code open. You will understand 70% of the codebase.

---

## The Full Picture

```
YOU
 │
 │  kubectl apply -f pod.yaml
 ▼
kubectl (CLI)
 │  HTTPS POST /api/v1/namespaces/default/pods
 ▼
kube-apiserver
 │  1. Authentication  — who are you?
 │  2. Authorization   — are you allowed?
 │  3. Admission       — mutate + validate
 │  4. Storage         — write to etcd
 ▼
etcd (durable storage)
 │  watch event emitted
 ▼
kube-scheduler (watching unscheduled pods)
 │  filter → score → bind
 │  PATCH pod.spec.nodeName = "worker-node-1"
 ▼
kube-apiserver (again)
 │  write nodeName to etcd
 │  watch event emitted
 ▼
kubelet on worker-node-1 (watching pods assigned to its node)
 │  SyncPod()
 │  gRPC CRI calls
 ▼
containerd / CRI-O
 │  runc
 ▼
Container running
 │
 └─► kubelet updates pod.status → API server → etcd
```

---

## Step 1 — kubectl Serializes and Sends the Request

**File:** `staging/src/k8s.io/kubectl/pkg/cmd/apply/apply.go`

```go
// ApplyOptions.Run() is the entry point for kubectl apply
func (o *ApplyOptions) Run() error {
    // 1. Read pod.yaml from disk
    // 2. Unmarshal YAML → runtime.Object
    // 3. GET the object to see if it exists
    // 4. If exists: send PATCH (server-side apply or strategic merge patch)
    //    If not:    send POST (create)
}
```

**What kubectl actually sends:**
```http
POST /api/v1/namespaces/default/pods HTTP/1.1
Host: kube-apiserver:6443
Authorization: Bearer <token>
Content-Type: application/json

{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": { "name": "my-pod", "namespace": "default" },
  "spec": { "containers": [{ "name": "app", "image": "nginx" }] }
}
```

**Key concept — Server-Side Apply:**
With `kubectl apply`, Kubernetes tracks which fields each manager "owns". If two actors both try to set the same field, there's a conflict. This prevents silent overwrites. The implementation is in `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/fieldmanager/`.

---

## Step 2 — kube-apiserver Receives the Request

**File:** `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/create.go`

The API server's HTTP handler:
1. Routes the URL path to the correct resource handler (pods in core group, v1)
2. Decodes the request body from JSON/YAML → internal Go type
3. Hands off to the chain: authentication → authorization → admission → storage

**URL routing:**
```
/api/v1/namespaces/{namespace}/pods  →  core/v1/pods handler
/apis/apps/v1/deployments            →  apps/v1/deployments handler
```

---

## Step 3 — Authentication

**File:** `staging/src/k8s.io/apiserver/pkg/authentication/`

The API server asks: **Who is making this request?**

Authentication methods (tried in order until one succeeds):
```
Client certificates    → extract Subject CN as username, O as groups
Bearer tokens          → service account tokens or OIDC tokens
Webhook auth           → delegate to an external service
Anonymous              → username = "system:anonymous", group = "system:unauthenticated"
```

Output of authentication: `UserInfo{ Name, Groups, Extra }`

**Service account tokens:**
When a Pod runs, it gets a token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This token authenticates the Pod to the API server. The kubelet refreshes it before expiry.

---

## Step 4 — Authorization

**File:** `staging/src/k8s.io/apiserver/pkg/authorization/`

The API server asks: **Is this user allowed to do this action on this resource?**

Authorization check: `(User, Verb, Resource, Namespace) → Allow/Deny`

Example: `(system:serviceaccount:default:my-app, create, pods, default) → Allow or Deny`

**RBAC** is the most common authorizer:
```yaml
# ClusterRole defines permissions
kind: ClusterRole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create"]

# RoleBinding grants them to a subject
kind: RoleBinding
subjects:
- kind: ServiceAccount
  name: my-app
roleRef:
  kind: ClusterRole
  name: pod-reader
```

If authorization fails → `403 Forbidden` returned to kubectl.

---

## Step 5 — Admission Controllers

**File:** `pkg/admission/` (built-in plugins) + `staging/src/k8s.io/apiserver/pkg/admission/`

Admission runs in two phases:

### Phase 1: Mutating Admission
Runs in sequence. Each plugin **can modify** the object. Used to:
- Set default values (DefaultStorageClass, DefaultTolerationSeconds)
- Inject sidecar containers (via MutatingWebhookConfiguration)
- Add labels or annotations

### Phase 2: Validating Admission
Runs in parallel. Each plugin **cannot modify** the object — only approve or reject. Used to:
- Enforce organizational policies
- Validate field constraints not expressible in JSON schema
- Resource quota enforcement

**External webhooks:**
`MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration` — the API server calls your HTTP endpoint during admission. This is how tools like OPA/Gatekeeper and Kyverno work.

---

## Step 6 — Validation

**File:** `pkg/apis/core/validation/validation.go`

After admission, the API server validates the object:
- Required fields present?
- Values in valid ranges?
- Names match DNS subdomain format?
- Port numbers in valid range?

If validation fails → `422 Unprocessable Entity` returned to kubectl.

---

## Step 7 — Write to etcd

**File:** `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go`

The object is:
1. Serialized to **protobuf** (not JSON — protobuf is smaller and faster)
2. Written to etcd at key: `/registry/pods/default/my-pod`
3. A watch event is emitted to all subscribers

The API server returns `201 Created` to kubectl.

```
kubectl output: pod/my-pod created
```

**`resourceVersion`:** When the object is written, etcd assigns it a revision number stored as `metadata.resourceVersion`. This is used for optimistic concurrency — if two components try to update the same object, the second one will see a conflict (HTTP 409) because the `resourceVersion` has changed.

---

## Step 8 — Scheduler Watches for Unscheduled Pods

**File:** `pkg/scheduler/scheduler.go` → `Run()`

The scheduler runs an informer that watches all Pods. When a Pod appears with `spec.nodeName == ""`, the informer puts it into the scheduling queue.

```go
// scheduler.go: Run()
func (sched *Scheduler) Run(ctx context.Context) {
    sched.SchedulingQueue.Run(logger)
    go wait.UntilWithContext(ctx, sched.ScheduleOne, 0)
    <-ctx.Done()
}
```

`ScheduleOne` runs in a goroutine loop:
```go
// sched.ScheduleOne pulls one pod from the queue and schedules it
func (sched *Scheduler) ScheduleOne(ctx context.Context) {
    podInfo, err := sched.NextEntity(logger)  // blocks until a pod is available
    // ... runs the scheduling pipeline
}
```

---

## Step 9 — The Scheduling Pipeline

**File:** `pkg/scheduler/schedule_one.go`

```
Pod enters queue
    │
    ▼  PreEnqueue plugins
Filter out pods that shouldn't be queued at all
    │
    ▼  PreFilter plugins
Compute per-pod data needed by filter plugins (cache results)
    │
    ▼  Filter plugins (run in parallel against all nodes)
NodeResourcesFit     — does the node have enough CPU/memory?
NodeAffinity         — does the node match pod's node affinity?
TaintToleration      — does the pod tolerate the node's taints?
PodTopologySpread    — does this placement satisfy spread constraints?
VolumeBinding        — can required volumes be bound on this node?
… (many more)
    │
    ▼  PostFilter plugins (only if no nodes pass Filter)
Preemption: can we evict lower-priority pods to make room?
    │
    ▼  Score plugins (rank remaining nodes 0–100)
LeastAllocated       — prefer nodes with more free resources
ImageLocality        — prefer nodes that already have the image
BalancedAllocation   — balance CPU/memory usage across nodes
    │
    ▼  NormalizeScore (scale scores to 0–100)
    │
    ▼  Reserve plugins
Reserve resources (e.g., claim a PVC binding)
    │
    ▼  Permit plugins
Gate the pod here temporarily (e.g., gang scheduling: wait for all pods in a group)
    │
    ▼  Bind plugins
Write pod.spec.nodeName = "worker-node-1" to the API server
```

After binding, the scheduler's responsibility is done. It moves to the next pod.

---

## Step 10 — Kubelet Detects the Pod

**File:** `pkg/kubelet/kubelet.go` → `syncLoop()`

The kubelet runs an informer that watches Pods filtered by `spec.nodeName == <this node's name>`. When the Pod appears with a nodeName set, the informer triggers:

```go
// kubelet.go syncLoop listens on multiple channels
func (kl *Kubelet) syncLoop(ctx context.Context, updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
    for {
        select {
        case update := <-updates:
            // pod was added/updated/deleted
            handler.HandlePodAdditions(update.Pods)
        }
    }
}
```

This puts the pod into the `podWorkers` — one goroutine per pod.

---

## Step 11 — Kubelet Syncs the Pod

**File:** `pkg/kubelet/kubelet_pods.go` → `SyncPod()`

`SyncPod` is the core kubelet reconciliation function:

```
SyncPod(pod)
    │
    ├── Create pod cgroup (resource limits)
    │
    ├── Make pod data directories (/var/lib/kubelet/pods/<uid>)
    │
    ├── Fetch Secrets and ConfigMaps referenced by the pod
    │
    ├── Mount volumes (via volumemanager)
    │
    ├── Pull images (if not already present)
    │
    ├── Create pod sandbox (RunPodSandbox CRI call)
    │   └── pause container — holds the network namespace
    │
    ├── Start init containers (one at a time, in order)
    │   └── each must succeed before the next starts
    │
    └── Start regular containers (in parallel)
        └── CreateContainer + StartContainer CRI calls
```

---

## Step 12 — CRI Calls to the Container Runtime

**Interface:** `staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto`

The Container Runtime Interface (CRI) is a gRPC API. Kubelet calls:

```go
// Create the pod sandbox (shared network namespace)
_, err = m.runtimeService.RunPodSandbox(ctx, podSandboxConfig, runtimeHandler)

// Pull the image
_, err = m.imageService.PullImage(ctx, &runtimeapi.ImageSpec{Image: image}, nil, podSandboxConfig)

// Create the container
containerID, err := m.runtimeService.CreateContainer(ctx, podSandboxID, containerConfig, podSandboxConfig)

// Start it
err = m.runtimeService.StartContainer(ctx, containerID)
```

`containerd` or `CRI-O` receives these gRPC calls and:
1. Calls `runc` with an OCI spec
2. `runc` creates: PID namespace, network namespace, mount namespace, cgroup, chroot
3. Your container process starts

---

## Step 13 — Status Flows Back

**File:** `pkg/kubelet/status/status_manager.go`

After starting containers, kubelet:
1. Updates its in-memory Pod status
2. The `StatusManager` periodically (or on change) PATCHes `pod.status` to the API server

```
pod.status.phase = "Running"
pod.status.containerStatuses[0].ready = true
pod.status.containerStatuses[0].state.running.startedAt = <timestamp>
pod.status.conditions[0].type = "Ready"
pod.status.conditions[0].status = "True"
```

**All done.** The Pod is running. If you run `kubectl get pod my-pod`, you see `Running`.

---

## What Happens If the Container Crashes?

1. Container runtime detects the exit
2. `PLEG` (Pod Lifecycle Event Generator) in kubelet detects the event from the runtime
3. `SyncPod` is triggered again
4. Kubelet checks `restartPolicy`:
   - `Always` → restart immediately (with exponential backoff)
   - `OnFailure` → restart only if exit code != 0
   - `Never` → don't restart
5. Container is restarted (or not)
6. Status updated

This is the kubelet's reconciliation loop in action — it continuously ensures actual container state matches desired pod spec.

---

## What Happens If a Node Dies?

1. `node-controller` in kube-controller-manager runs a health check loop
2. After ~40 seconds of no heartbeat, node is marked `NotReady`
3. After ~5 minutes, `node-controller` taints the node with `node.kubernetes.io/unreachable`
4. Pods with a toleration timeout on that taint are evicted
5. Eviction means: delete the Pod objects from etcd
6. ReplicaSet controller sees pod count < desired count
7. Creates new Pod objects
8. Scheduler assigns them to healthy nodes
9. Kubelet on healthy node starts the containers

The dead node's kubelet eventually reconnects and reconciles its actual state.

---

## Key Interfaces in This Flow

| Interface | File | Purpose |
|-----------|------|---------|
| `authenticator.Request` | `staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go` | Authenticate a request |
| `authorizer.Authorizer` | `staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go` | Authorize a request |
| `admission.Interface` | `staging/src/k8s.io/apiserver/pkg/admission/interfaces.go` | Admission plugin |
| `storage.Interface` | `staging/src/k8s.io/apiserver/pkg/storage/interfaces.go` | Read/write objects |
| `framework.FilterPlugin` | `pkg/scheduler/framework/interface.go` | Scheduler filter |
| `framework.ScorePlugin` | `pkg/scheduler/framework/interface.go` | Scheduler score |
| `runtimeapi.RuntimeServiceClient` | `staging/src/k8s.io/cri-api/...` | CRI gRPC interface |
