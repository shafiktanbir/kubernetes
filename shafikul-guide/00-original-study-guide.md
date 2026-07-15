# Kubernetes Codebase Study Guide
> A grounded reading guide tied to this actual repository.
> Read top-to-bottom once, then use as a reference when navigating.

---

## Table of Contents

1. [Mental Model — How to Think About This Codebase](#1-mental-model)
2. [Repository Map — What Every Directory Does](#2-repository-map)
3. [The 7 Main Components](#3-the-7-main-components)
4. [How Kubernetes Actually Works (Reconciliation Loop)](#4-how-kubernetes-actually-works)
5. [Full Request Flow: kubectl apply -f pod.yaml](#5-full-request-flow)
6. [Code Architecture Patterns You Will See Everywhere](#6-code-architecture-patterns)
7. [Key Libraries You Must Know](#7-key-libraries-you-must-know)
8. [How to Read Any Package (Step-by-Step)](#8-how-to-read-any-package)
9. [Pick One Subsystem — Guide for Each](#9-pick-one-subsystem)
10. [Running Kubernetes Locally](#10-running-kubernetes-locally)
11. [Before Your First PR — Checklist](#11-before-your-first-pr)
12. [Common Questions to Ask Me](#12-common-questions-to-ask-me)

---

## 1. Mental Model

Before reading a single file, burn these four ideas into your head:

### Kubernetes is a control loop machine

Every component does this:
```
while true:
    observe current state
    compare to desired state
    take action to close the gap
    sleep / wait for event
```
This is called **reconciliation**. Nothing is pushed. Everything is pulled and compared.

### State lives in exactly one place: etcd

All other components are stateless and derive everything from what they read from the API server (which reads from etcd). If etcd disappears, the cluster brain dies but running containers keep running.

### Components communicate through the API server, never directly

```
kubelet ──────────► API Server ◄─────── scheduler
controller ──────► API Server ◄─────── kubectl
```
Nobody talks to etcd except the API server. Nobody talks to kubelet except through the API server's watch mechanism.

### Interfaces over implementations

Kubernetes uses Go interfaces extensively. A function rarely accepts a concrete struct. It accepts an interface. This lets the same scheduler code work with real nodes in production and fake nodes in tests. When reading code, **find the interface first, then find the implementation**.

---

## 2. Repository Map

```
kubernetes/
├── cmd/                    # Binary entry points (thin main.go files)
├── pkg/                    # Internal libraries (not published)
├── staging/                # Published sub-modules (client-go, apimachinery, etc.)
│   └── src/k8s.io/
├── api/                    # OpenAPI JSON specs and validation rules (NOT Go source)
├── test/                   # All test code
├── hack/                   # Developer scripts (generators, verifiers, local cluster)
├── build/                  # Container image and release scripts
└── vendor/                 # Vendored third-party dependencies
```

### cmd/ — Binary Entry Points

Each subdirectory is one binary. The pattern is always the same:

```
cmd/kube-apiserver/
    apiserver.go        ← main() — 5 lines, calls app.NewAPIServerCommand()
    app/
        server.go       ← Real initialization, flags, startup logic

cmd/kubelet/
    kubelet.go          ← main() — 5 lines, calls app.NewKubeletCommand()
    app/
        server.go       ← Real initialization

cmd/kubectl/
cmd/kube-scheduler/
cmd/kube-controller-manager/
cmd/kube-proxy/
```

**Rule: Never start reading `main.go`. Jump straight to `cmd/<binary>/app/server.go`.**

What's inside `cmd/`:
- `kube-apiserver/` — The REST API gateway
- `kube-scheduler/` — Pod placement
- `kube-controller-manager/` — All reconciliation controllers
- `kubelet/` — Node agent
- `kube-proxy/` — Network routing on each node
- `kubectl/` — CLI client
- `kubeadm/` — Cluster bootstrapper
- Many code generation tools (`gendocs`, `genman`, etc.)

### pkg/ — Internal Libraries

Shared code that lives inside `k8s.io/kubernetes` itself but is not published as a standalone module. This is where the **actual implementation** of each component lives.

Key packages:
```
pkg/
├── scheduler/          ← Scheduler logic (filter, score, bind)
├── kubelet/            ← Kubelet node agent logic
├── controller/         ← All controller implementations (deployment, replicaset, job, etc.)
├── kubeapiserver/      ← API server internals
├── admission/          ← Admission controller plugins
├── proxy/              ← kube-proxy networking logic
├── kubectl/            ← kubectl command implementations
├── apis/               ← Internal Go type definitions
└── registry/           ← Storage layer (how objects are stored/retrieved)
```

### staging/ — Published Sub-Modules

This is the most confusing part. Here is the reality:

- `staging/src/k8s.io/client-go/` is the **authoritative source** of the `client-go` repo.
- A publishing bot periodically copies this to `github.com/kubernetes/client-go`.
- When you import `k8s.io/client-go/kubernetes` in Go code, it resolves here via `go.work`.

**You can and should edit files in `staging/`. It is not generated.**

Key staged modules:
```
staging/src/k8s.io/
├── api/                    ← Go type definitions for all API objects (Pod, Deployment, etc.)
├── apimachinery/           ← Core machinery: ObjectMeta, TypeMeta, serialization, errors
├── apiserver/              ← Generic API server framework
├── client-go/              ← The official Go client library
├── controller-manager/     ← Controller manager framework
├── kube-scheduler/         ← Scheduler plugin interface definitions
├── kubectl/                ← kubectl library code
├── kubelet/                ← Kubelet public API types
├── cri-api/                ← Container Runtime Interface (gRPC definitions)
└── code-generator/         ← Tools to generate clientsets, informers, listers
```

### api/ — OpenAPI Specs

```
api/
├── openapi-spec/           ← swagger.json / openapi.json (generated, do not edit)
└── api-rules/              ← Validation rules for API review
```
This is **not Go source code**. It is metadata used by docs generators and the API linter. When you add a new API field, the spec here is regenerated via `make update`.

### test/ — All Tests

```
test/
├── e2e/                    ← End-to-end tests (require a running cluster)
├── e2e_node/               ← Node-level e2e tests
├── integration/            ← Integration tests (API server + etcd, no full cluster)
├── conformance/            ← Kubernetes conformance tests
└── fixtures/               ← Test data and manifests
```

For contributors, **integration tests** are the most important. They test real API server behavior without needing a full cluster.

### hack/ — Developer Scripts

This directory runs your world as a contributor. Key scripts:

| Script | What it does |
|--------|-------------|
| `hack/local-up-cluster.sh` | Start a full local cluster in one command |
| `hack/update-codegen.sh` | Regenerate clients, informers, listers |
| `hack/update-all.sh` | Run ALL generators and formatters |
| `hack/verify-all.sh` | Run ALL verification checks |
| `hack/pin-dependency.sh` | Pin a vendored dependency to a version |
| `hack/update-vendor.sh` | Update vendor/ after dependency changes |
| `hack/boilerplate/boilerplate.go.txt` | License header required on every `.go` file |

**Rule: Never run `go mod tidy`. Use `hack/pin-dependency.sh` + `hack/update-vendor.sh`.**

### build/ — Release Scripts

Container image builds, cross-compilation, release packaging. You will rarely touch this as a new contributor.

### vendor/ — Third-Party Dependencies

Checked-in copy of all external dependencies. Managed by `hack/update-vendor.sh`. Contains things like etcd client, gRPC, Prometheus, etc.

---

## 3. The 7 Main Components

### kube-apiserver

**What it does:** The single front door of the cluster. All cluster state changes go through it. It validates requests, enforces policy, writes to etcd, and notifies watchers.

**Where the code lives:**
```
cmd/kube-apiserver/app/server.go    ← startup
pkg/kubeapiserver/                  ← API server internals
staging/src/k8s.io/apiserver/       ← generic apiserver framework
pkg/registry/                       ← storage layer per resource type
```

**What it does NOT do:** It does not schedule pods. It does not run containers. It is a REST API + database gateway.

---

### etcd

**What it does:** A distributed key-value store. Stores all cluster state as serialized protobuf. The only persistent storage in Kubernetes.

**Where the code lives:** Not in this repo. External dependency at `vendor/go.etcd.io/etcd/`.

**Key fact:** kube-apiserver is the **only** component that writes to etcd. Everything else goes through the API server.

---

### kube-scheduler

**What it does:** Watches for Pods with no `spec.nodeName` set. Runs them through a pipeline of plugins (filter → score → bind) and writes the chosen node name back to the API server.

**Where the code lives:**
```
cmd/kube-scheduler/app/server.go    ← startup
pkg/scheduler/scheduler.go         ← Scheduler struct + Run() + New()
pkg/scheduler/schedule_one.go      ← ScheduleOne() — the main scheduling loop
pkg/scheduler/framework/
    interface.go                    ← Framework interface (plugin extension points)
    types.go                        ← NodeToStatus, CycleState types
pkg/scheduler/framework/plugins/   ← Concrete plugin implementations
pkg/scheduler/backend/queue/       ← Priority queue for unscheduled pods
pkg/scheduler/backend/cache/       ← Node/Pod cache (avoid calling API server on every decision)
```

**The scheduling pipeline (read this order):**
```
Scheduler.Run()
  └─► Scheduler.ScheduleOne()     (per pod, in a goroutine loop)
        ├─ PreFilter plugins
        ├─ Filter plugins          (which nodes can run this pod?)
        ├─ PostFilter plugins      (preemption if no nodes pass)
        ├─ Score plugins           (rank remaining nodes)
        ├─ Normalize scores
        ├─ Select winner
        ├─ Reserve plugins
        ├─ Permit plugins
        └─ Bind plugins            (write nodeName to API server)
```

---

### kube-controller-manager

**What it does:** Runs dozens of control loops in a single binary. Each controller watches specific resources and reconciles actual state toward desired state.

**Where the code lives:**
```
cmd/kube-controller-manager/       ← startup
pkg/controller/
    deployment/                    ← Deployment controller
    replicaset/                    ← ReplicaSet controller
    job/                           ← Job controller
    daemon/                        ← DaemonSet controller
    statefulset/                   ← StatefulSet controller
    garbagecollector/              ← GC controller
    nodelifecycle/                 ← Node health controller
    namespace/                     ← Namespace cleanup
    serviceaccount/                ← ServiceAccount token creation
    ... (38 total controllers)
```

**The reconciliation pattern (same in every controller):**
```go
// Every controller looks like this:
func (c *Controller) Run(ctx context.Context) {
    go wait.UntilWithContext(ctx, c.runWorker, time.Second)
}

func (c *Controller) runWorker(ctx context.Context) {
    for c.processNextItem(ctx) {}
}

func (c *Controller) processNextItem(ctx context.Context) bool {
    key, quit := c.queue.Get()
    // ... get object from lister (cache) ...
    // ... compare actual vs desired ...
    // ... call API server to fix the diff ...
}
```

---

### kubelet

**What it does:** Node-level pod manager. Runs on every worker node. Watches the API server for pods assigned to its node, then drives the container runtime (via CRI gRPC calls) to start, stop, and monitor containers. Reports node and pod status back.

**Where the code lives:**
```
cmd/kubelet/app/server.go          ← startup
pkg/kubelet/
    kubelet.go                     ← Kubelet struct (152KB — the main file)
    kubelet_pods.go                ← Pod lifecycle management (122KB)
    pod_workers.go                 ← Per-pod goroutine management (78KB)
    kuberuntime/                   ← CRI calls (create/start/stop containers)
    pleg/                          ← Pod Lifecycle Event Generator (watches runtime state)
    eviction/                      ← Memory/disk pressure eviction logic
    volumemanager/                 ← Mounts/unmounts volumes for pods
    prober/                        ← Liveness/readiness/startup probes
    status/                        ← Updates pod status back to API server
staging/src/k8s.io/cri-api/       ← gRPC interface to container runtime
```

**The kubelet sync loop:**
```
kubelet starts
  ├─ PLEG: watches container runtime events (container started/stopped/died)
  ├─ SyncLoop: main reconciliation goroutine
  │     every second (or on event):
  │       for each pod assigned to this node:
  │           SyncPod() → ensure containers are in desired state
  │               ├─ pull image if needed
  │               ├─ create sandbox (pause container)
  │               ├─ start init containers
  │               └─ start regular containers
  └─ StatusManager: updates Pod.Status → API server
```

---

### kube-proxy

**What it does:** Runs on every node. Programs iptables or IPVS rules so that a Service's ClusterIP (virtual IP) routes to the correct backend Pods. When a Pod sends a request to `10.96.0.1:80` (a Service IP), kube-proxy's rules transparently redirect it to a real Pod IP.

**Where the code lives:**
```
cmd/kube-proxy/                    ← startup
pkg/proxy/                         ← proxy logic
    iptables/                      ← iptables mode
    ipvs/                          ← IPVS mode
    winkernel/                     ← Windows mode
staging/src/k8s.io/kube-proxy/     ← kube-proxy API types
```

---

### kubectl

**What it does:** CLI client. Parses user commands, serializes YAML/JSON, and makes REST calls to kube-apiserver. Has no special authority — it is just a very smart HTTP client.

**Where the code lives:**
```
cmd/kubectl/                       ← entry point
staging/src/k8s.io/kubectl/
    pkg/cmd/                       ← Every subcommand (apply, get, describe, logs, exec...)
    pkg/cmd/apply/apply.go         ← kubectl apply implementation
    pkg/cmd/get/get.go             ← kubectl get implementation
```

---

## 4. How Kubernetes Actually Works

The reconciliation loop in plain English:

1. **You declare desired state** via `kubectl apply -f pod.yaml`.
2. **API server stores it** in etcd and notifies all watchers.
3. **Scheduler** (watching for unscheduled pods) picks a node and writes `pod.spec.nodeName`.
4. **Kubelet on that node** (watching for pods assigned to it) calls the container runtime to start the container.
5. **Container runtime** (containerd/CRI-O) uses runc to create the container.
6. **Kubelet** updates `pod.status` back to API server.
7. **Controller manager** watches actual state (pod count) and desired state (replica count) and creates/deletes pods to reconcile them.

This loop runs continuously. Kubernetes never "finishes" — it permanently reconciles.

---

## 5. Full Request Flow

### `kubectl apply -f pod.yaml`

```
Step 1 — kubectl
  File: staging/src/k8s.io/kubectl/pkg/cmd/apply/apply.go
  - Reads pod.yaml, unmarshals YAML → Go struct
  - Detects if object exists (GET first)
  - Sends POST (create) or PATCH (update) to API server
  - URL: POST /api/v1/namespaces/default/pods

Step 2 — kube-apiserver receives request
  File: staging/src/k8s.io/apiserver/pkg/endpoints/handlers/create.go
  - Parses HTTP request, identifies resource type and verb

Step 3 — Authentication
  File: staging/src/k8s.io/apiserver/pkg/authentication/
  - Who is making this request?
  - Methods: client certificates, bearer tokens, webhook auth
  - Output: UserInfo{Name, Groups, Extra}

Step 4 — Authorization
  File: staging/src/k8s.io/apiserver/pkg/authorization/
  - Is this user allowed to CREATE pods in namespace "default"?
  - RBAC check: does user have a ClusterRole/Role binding that allows this?
  - If denied → 403 Forbidden

Step 5 — Admission Controllers
  File: pkg/admission/
  - Mutating admission: can modify the object (e.g., inject sidecars, set defaults)
  - Validating admission: can reject the object (e.g., policy enforcement)
  - Also runs external webhook admission controllers

Step 6 — Validation
  - Schema validation: are all required fields present?
  - Field validation: are values in allowed range?
  - API rules validation

Step 7 — Write to etcd
  File: pkg/registry/core/pod/storage/
  - Object serialized to protobuf
  - Written to etcd key: /registry/pods/default/<pod-name>
  - Watch event emitted to all watchers
  - API server returns 201 Created to kubectl

--- kubectl gets 201, prints "pod/my-pod created" ---

Step 8 — Scheduler watches for unscheduled pod
  File: pkg/scheduler/scheduler.go → scheduler.Run()
  - Informer triggers: new Pod with spec.nodeName="" added to scheduling queue
  - ScheduleOne() pops the pod from queue

Step 9 — Scheduling pipeline runs
  File: pkg/scheduler/schedule_one.go
  - PreFilter: check pod resource requirements
  - Filter: eliminate nodes that can't run this pod (resources, taints, affinity)
  - Score: rank remaining nodes (least allocated, image locality, etc.)
  - Bind: write spec.nodeName = "worker-node-1" via API server PATCH

--- Pod object now has spec.nodeName set ---

Step 10 — Kubelet on worker-node-1 sees the pod
  File: pkg/kubelet/kubelet.go → SyncLoop()
  - Informer triggers: pod assigned to this node, sync it
  - SyncPod() called

Step 11 — Kubelet drives container runtime
  File: pkg/kubelet/kuberuntime/
  - Calls CRI gRPC: RunPodSandbox (create network namespace, "pause" container)
  - Calls CRI gRPC: PullImage
  - Calls CRI gRPC: CreateContainer + StartContainer

Step 12 — Container runtime (containerd/CRI-O) starts the container
  - Uses runc (OCI runtime) to create Linux namespaces, cgroups
  - Container is now running

Step 13 — Kubelet updates pod status
  File: pkg/kubelet/status/status_manager.go
  - PATCH pod.status.phase = "Running"
  - PATCH pod.status.containerStatuses = [{ready: true, ...}]
  - API server writes update to etcd

--- Done. Container is running. ---
```

---

## 6. Code Architecture Patterns

### Pattern 1: Options / Functional Options

You will see this constantly in Kubernetes:

```go
// Instead of a massive constructor with 20 arguments:
func New(ctx context.Context, client clientset.Interface, opts ...Option) (*Scheduler, error) {
    options := defaultSchedulerOptions   // sensible defaults
    for _, opt := range opts {
        opt(&options)                    // each Option modifies the defaults
    }
    // ...
}

// Usage:
sched, err := New(ctx, client,
    WithParallelism(32),
    WithPercentageOfNodesToScore(&pct),
)
```

This pattern is in `pkg/scheduler/scheduler.go` and almost every other constructor.

### Pattern 2: Informer + WorkQueue (The Controller Pattern)

Every controller uses this:

```go
// 1. Informer: watches API server, maintains local cache
podInformer := informerFactory.Core().V1().Pods()

// 2. Event handler: puts work onto a queue (never processes inline)
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { queue.Add(key) },
    UpdateFunc: func(old, new interface{}) { queue.Add(key) },
    DeleteFunc: func(obj interface{}) { queue.Add(key) },
})

// 3. Worker: dequeues and reconciles
func (c *Controller) processNextItem() {
    key, _ := c.queue.Get()
    obj, _ := c.lister.Get(key)  // read from local cache, not API server
    // reconcile...
    c.queue.Done(key)
}
```

**Why queue instead of processing inline?** The event handler runs on the informer's goroutine. You must not block it. The queue decouples event detection from processing and provides retry logic.

### Pattern 3: Interface-First Design

```go
// In staging/src/k8s.io/apiserver/pkg/storage/interfaces.go:
type Interface interface {
    Get(ctx, key string, opts GetOptions, objPtr runtime.Object) error
    Create(ctx, key string, obj, out runtime.Object, ttl uint64) error
    // ...
}

// Real implementation: etcd3.store (in pkg/)
// Test implementation: testing/fake_storage.go
// Both satisfy the same Interface — tests are fast, production is real
```

### Pattern 4: Plugin / Registry Pattern (Scheduler)

```go
// Each scheduler plugin implements one or more interfaces:
type FilterPlugin interface {
    Plugin                          // Name() string
    Filter(ctx, state CycleState, pod *v1.Pod, nodeInfo NodeInfo) *Status
}

// Plugins register themselves:
registry := frameworkplugins.NewInTreeRegistry()
// registry["NodeResourcesFit"] = NewNodeResourcesFit
// registry["NodeAffinity"] = NewNodeAffinity
// ...

// Framework instantiates plugins from registry based on config
```

### Pattern 5: Feature Gates

Features are toggled at runtime without recompilation:

```go
if feature.DefaultFeatureGate.Enabled(features.DynamicResourceAllocation) {
    // new code path
}
```

Feature gates let the project ship experimental features safely. When reading code, you will often see two code paths — one for `if featureGate.Enabled(...)` and one for the fallback.

---

## 7. Key Libraries You Must Know

### k8s.io/client-go (`staging/src/k8s.io/client-go/`)

The official Go client. Everything uses it.

```
client-go/
├── kubernetes/             ← Typed clientset (client.CoreV1().Pods().Get(...))
├── dynamic/                ← Dynamic client for unknown/CRD types
├── tools/cache/            ← Informers, Listers, WorkQueues ← READ THIS FIRST
├── tools/clientcmd/        ← Loads kubeconfig files
├── rest/                   ← Low-level HTTP client
└── informers/              ← SharedInformerFactory
```

**The most important sub-package:** `tools/cache` — this is the foundation of every controller in Kubernetes.

### k8s.io/apimachinery (`staging/src/k8s.io/apimachinery/`)

Core Kubernetes types and machinery. You cannot avoid this.

```
apimachinery/pkg/
├── api/errors/             ← IsNotFound(), IsConflict(), etc.
├── apis/meta/v1/           ← ObjectMeta, TypeMeta, ListMeta
├── runtime/                ← runtime.Object interface, serialization
├── util/wait/              ← wait.Until(), wait.PollUntilContextTimeout()
└── util/sets/              ← String/Int sets
```

### k8s.io/api (`staging/src/k8s.io/api/`)

The Go type definitions for every Kubernetes API object.

```go
import corev1 "k8s.io/api/core/v1"
// corev1.Pod, corev1.Node, corev1.Service, corev1.ConfigMap...

import appsv1 "k8s.io/api/apps/v1"
// appsv1.Deployment, appsv1.ReplicaSet, appsv1.StatefulSet...
```

### k8s.io/klog (`vendor/k8s.io/klog/`)

Kubernetes logging library. All log calls look like:

```go
klog.V(4).InfoS("Scheduling pod", "pod", klog.KObj(pod), "node", nodeName)
klog.ErrorS(err, "Failed to sync pod", "pod", klog.KObj(pod))
```

`V(N)` is verbosity level. Higher = more verbose. Default log level is 0.

---

## 8. How to Read Any Package

Follow this order every time:

```
Step 1: Read README.md (if it exists)
Step 2: Read doc.go (package-level documentation)
Step 3: Find the main struct/type
          grep -r "type .* struct" --include="*.go" | grep -v test
Step 4: Find the constructor
          Usually named New(), NewXxx(), or Build()
Step 5: Find the main method / entry point
          Usually Run(ctx), Start(ctx), or Reconcile()
Step 6: Follow the call graph DOWN from that entry point
          Don't read every file — follow execution
Step 7: Find the interfaces
          type XxxInterface interface { ... }
          Read interface before implementation
```

### Example: Reading pkg/scheduler/

```
1. No README — skip
2. No doc.go in pkg/scheduler/ — go to step 3
3. Main struct: type Scheduler struct  (in scheduler.go line 67)
4. Constructor: func New(ctx, client, ...) (*Scheduler, error)  (line 281)
5. Entry point: func (sched *Scheduler) Run(ctx context.Context)  (line 525)
6. Run() calls:
     sched.SchedulingQueue.Run()     → pkg/scheduler/backend/queue/
     go wait.UntilWithContext(ctx, sched.ScheduleOne, 0)
7. ScheduleOne is in schedule_one.go
   Follow that file to understand the full scheduling pipeline.
```

---

## 9. Pick One Subsystem

### Option A: kubectl (Best for beginners)

**Why:** Self-contained, familiar (you already use it), clear input→output, excellent test coverage, no cluster needed.

**Entry path:**
```
staging/src/k8s.io/kubectl/pkg/cmd/
    apply/apply.go          ← Start here (kubectl apply)
    get/get.go              ← Then here (kubectl get)
    describe/describe.go
```

**Good first issues:** Improve error messages, fix help text, add missing output columns, fix edge cases in `--dry-run` handling.

**How to test:**
```bash
make test WHAT=./staging/src/k8s.io/kubectl/...
```

---

### Option B: Scheduler (Best for algorithmic work)

**Why:** Plugin system makes it easy to add one plugin at a time. Framework is well-documented. Clear interfaces.

**Entry path:**
```
pkg/scheduler/framework/interface.go    ← Plugin interfaces (start here)
pkg/scheduler/scheduler.go              ← Scheduler struct + New() + Run()
pkg/scheduler/schedule_one.go          ← ScheduleOne() — the scheduling loop
pkg/scheduler/framework/plugins/        ← Concrete plugins (read 2-3)
    noderesources/fit.go               ← NodeResourcesFit plugin (simple, good first read)
    nodeaffinity/node_affinity.go      ← NodeAffinity plugin
```

**Good first issues:** Improve logging, add metrics, fix plugin edge cases.

**How to test:**
```bash
make test WHAT=./pkg/scheduler/...
```

---

### Option C: Kubelet (Most active area, real workload behavior)

**Why:** Most community activity, impacts real workloads, many good-first-issues.

**Entry path:**
```
pkg/kubelet/
    kubelet.go              ← Kubelet struct (big file, read the first 200 lines)
    pod_workers.go          ← Per-pod goroutine management
    kubelet_pods.go         ← SyncPod() — the core pod sync
    kuberuntime/            ← CRI calls
    prober/                 ← Liveness/readiness probe logic
    eviction/               ← Eviction logic
```

**Warning:** `kubelet.go` is 152,000 bytes. Don't try to read it all. Use Go to Definition to navigate.

**How to test:**
```bash
make test WHAT=./pkg/kubelet/...
```

---

### Option D: API Server (Deep protocol/serialization work)

**Why:** Core infrastructure, affects all other components.

**Entry path:**
```
staging/src/k8s.io/apiserver/
    pkg/endpoints/handlers/create.go   ← How POST /pods is handled
    pkg/authentication/                ← Who you are
    pkg/authorization/                 ← What you can do
    pkg/admission/                     ← Policy enforcement
    pkg/registry/                      ← How objects are stored
pkg/registry/core/pod/storage/        ← Pod-specific storage
```

---

## 10. Running Kubernetes Locally

### Start a local cluster

```bash
# Start everything: API server, scheduler, controller-manager, kubelet, etcd
hack/local-up-cluster.sh
```

This runs a single-node cluster. You don't need Docker. You do need `etcd` installed (`hack/install-etcd.sh`).

### Run unit tests

```bash
# Test one package
make test WHAT=./pkg/scheduler/... GOFLAGS=-v

# Test multiple packages
make test WHAT=./pkg/kubelet/... GOFLAGS=-v
```

### Run integration tests

```bash
# Integration tests (need no full cluster, just API server + etcd)
make test-integration WHAT=./test/integration/scheduler/...
```

### Verify your changes

```bash
# Run ALL checks (linting, formatting, generated code, imports, etc.)
make verify

# Run just formatting
hack/verify-gofmt.sh

# Run just linting
hack/verify-golangci-lint.sh
```

### Add temporary logging

```go
import "k8s.io/klog/v2"

// Add this anywhere:
klog.V(4).InfoS("Debug: entering SyncPod", "pod", klog.KObj(pod))
```

Run with: `./hack/local-up-cluster.sh --v=4`

**Always remove debug logging before submitting a PR.**

---

## 11. Before Your First PR — Checklist

### Finding Issues

- GitHub label: `good first issue` — https://github.com/kubernetes/kubernetes/issues?q=label%3A%22good+first+issue%22
- GitHub label: `help wanted`
- GitHub label: `kind/bug` + the subsystem you chose

### Rules Every Contributor Must Follow

- [ ] **License header** on every `.go` file — copy from `hack/boilerplate/boilerplate.go.txt`
- [ ] **Never hand-edit** `zz_generated.*` or `generated.pb.go` — run `make update` instead
- [ ] **Never run `go mod tidy`** — use `hack/pin-dependency.sh` + `hack/update-vendor.sh`
- [ ] **No `@mentions`** in commit messages (e.g., `@reviewer`)
- [ ] **No `fixes #123`** keywords in commit messages
- [ ] **No `Co-authored-by:`** in commit messages
- [ ] Run `make verify` before pushing — CI will catch it anyway, save yourself a round-trip
- [ ] Add or update relevant tests for every change
- [ ] Keep changes focused — one thing per PR

### Package Names

- Lowercase, single word, matches the directory name
- Example: directory `pkg/scheduler` → `package scheduler`

### Commit Message Format

```
component: short description of what changed

Longer explanation of why this change was made.
What problem does it solve? What is the reasoning?
```

Examples:
```
scheduler: fix panic when node has nil capacity

kubelet: improve error message when CRI sandbox creation fails

kubectl: add --output=name flag to wait command
```

### OWNERS Files

Every directory has an `OWNERS` file. PRs must be approved by someone listed as an approver for the changed files. Find the right approver before asking for review.

---

## 12. Common Questions to Ask Me

Use this guide to study, then come back and ask questions like:

**On architecture:**
- "Explain how the scheduler filter plugins work with a concrete example"
- "What is the difference between a Lister and an Informer?"
- "How does server-side apply differ from client-side apply?"
- "Why does the kubelet use a PodWorker goroutine per pod?"

**On a specific file/function:**
- "Walk me through what `SyncPod()` does step by step"
- "What happens in `ScheduleOne()` if no nodes pass the filter phase?"
- "How does the Deployment controller decide when to create a new ReplicaSet?"

**On contributor workflow:**
- "What files do I need to change to add a new scheduler plugin?"
- "What tests do I need to write for a new kubelet feature?"
- "How do I regenerate the client after adding a new API field?"

**On a specific subsystem:**
- "Show me how a pod gets evicted when a node is under memory pressure"
- "How does RBAC authorization work in the API server?"
- "What is the purpose of the `SchedulingGates` feature?"

---

*This guide is tied to the Kubernetes repo at `/home/shafikul/Documents/coding/kubernetes`.*
*Last updated: July 2026.*
