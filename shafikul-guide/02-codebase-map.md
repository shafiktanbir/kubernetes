# 02 — Codebase Map
> A directory-by-directory breakdown. Reference when you don't know where something lives.

---

## Repository Root Layout

```
kubernetes/
├── cmd/          Binary entry points. One directory per binary.
├── pkg/          Internal libraries. Used by this repo only, not published.
├── staging/      Published sub-modules (client-go, apimachinery, apiserver…)
├── api/          OpenAPI JSON specs and API validation rules. NOT Go source.
├── test/         All test code: unit helpers, integration, e2e, conformance.
├── hack/         Developer scripts: codegen, verify, local cluster startup.
├── build/        Container image builds and release scripts.
├── vendor/       Vendored third-party dependencies.
├── plugin/       Out-of-tree plugin support code.
└── cluster/      Cluster bring-up scripts (older, mostly deprecated).
```

---

## cmd/ — Binary Entry Points

**Rule: Every `main.go` here is ~5 lines. Real logic is always in `cmd/<binary>/app/`.**

```
cmd/
├── kube-apiserver/         REST API gateway
│   ├── apiserver.go        main() → app.NewAPIServerCommand() → cli.Run()
│   └── app/
│       ├── server.go       Real initialization — start here
│       └── options/        Flag definitions
│
├── kube-scheduler/         Pod placement engine
│   ├── scheduler.go        main() → app.NewSchedulerCommand()
│   └── app/
│       └── server.go       Real initialization
│
├── kube-controller-manager/ All reconciliation controllers
│   ├── controller-manager.go
│   └── app/
│       └── controllermanager.go
│
├── kubelet/                Node agent
│   ├── kubelet.go
│   └── app/
│       └── server.go
│
├── kube-proxy/             Node network routing
│   ├── proxy.go
│   └── app/
│       └── server.go
│
├── kubectl/                CLI client
│   └── kubectl.go
│
├── kubeadm/                Cluster bootstrapper
│
└── (many codegen tools: gendocs, genman, genyaml, genkubedocs…)
```

---

## pkg/ — Internal Libraries

This is where the actual implementation lives. `cmd/` just wires it up.

```
pkg/
├── scheduler/              Scheduler implementation
│   ├── scheduler.go        Scheduler struct, New(), Run()
│   ├── schedule_one.go     ScheduleOne() — the per-pod scheduling loop
│   ├── eventhandlers.go    Informer event handlers (what re-triggers scheduling)
│   ├── framework/
│   │   ├── interface.go    Framework interface — plugin extension points
│   │   ├── types.go        NodeToStatus, CycleState, NodeInfo types
│   │   └── plugins/        Concrete plugin implementations
│   └── backend/
│       ├── queue/          Priority queue for unscheduled pods
│       └── cache/          Node/Pod cache (local copy to avoid API calls)
│
├── kubelet/                Kubelet implementation
│   ├── kubelet.go          Kubelet struct (152KB — huge, navigate don't read linearly)
│   ├── kubelet_pods.go     SyncPod(), pod lifecycle (122KB)
│   ├── pod_workers.go      Per-pod goroutine management (78KB)
│   ├── kuberuntime/        CRI calls — creating/starting/stopping containers
│   ├── pleg/               Pod Lifecycle Event Generator
│   ├── eviction/           Eviction manager (memory/disk pressure)
│   ├── volumemanager/      Volume mount/unmount lifecycle
│   ├── prober/             Liveness, readiness, startup probes
│   └── status/             Updates pod.status back to API server
│
├── controller/             All controller implementations
│   ├── deployment/         Deployment controller
│   ├── replicaset/         ReplicaSet controller
│   ├── job/                Job controller
│   ├── daemon/             DaemonSet controller
│   ├── statefulset/        StatefulSet controller
│   ├── garbagecollector/   GC controller (orphan cleanup via owner references)
│   ├── nodelifecycle/      Node health + taint/eviction
│   ├── namespace/          Namespace deletion cleanup
│   ├── serviceaccount/     Token creation for service accounts
│   └── … (38 controllers total)
│
├── kubeapiserver/          API server internals (on top of generic apiserver)
│   └── admission/          Built-in admission plugins
│
├── admission/              Admission plugin implementations
│
├── registry/               Storage layer — how each resource type is persisted
│   ├── core/pod/           Pod storage
│   ├── core/node/          Node storage
│   ├── apps/deployment/    Deployment storage
│   └── …
│
├── proxy/                  kube-proxy network programming
│   ├── iptables/           iptables backend
│   └── ipvs/               IPVS backend
│
├── apis/                   Internal Go type definitions (pre-conversion types)
│   ├── core/               Pod, Node, Service, etc.
│   └── apps/               Deployment, ReplicaSet, etc.
│
└── kubectl/                kubectl command implementations (internal)
```

---

## staging/ — Published Sub-Modules

The most confusing part. Here is the reality:

- `staging/src/k8s.io/client-go/` is **the actual source** of `github.com/kubernetes/client-go`
- A publishing bot syncs it to the separate GitHub repo automatically
- When Go code imports `k8s.io/client-go/...`, `go.work` resolves it here
- **You can and should edit files in `staging/`. They are not generated.**

```
staging/src/k8s.io/
│
├── api/                    Go type definitions for all API objects
│   ├── core/v1/            Pod, Node, Service, ConfigMap, Secret, PV, PVC…
│   ├── apps/v1/            Deployment, ReplicaSet, StatefulSet, DaemonSet…
│   ├── batch/v1/           Job, CronJob
│   └── rbac/v1/            ClusterRole, RoleBinding…
│
├── apimachinery/           Core Kubernetes machinery
│   ├── pkg/api/errors/     IsNotFound(), IsConflict(), IsAlreadyExists()…
│   ├── pkg/apis/meta/v1/   ObjectMeta, TypeMeta, ListMeta
│   ├── pkg/runtime/        runtime.Object interface, serialization
│   └── pkg/util/wait/      wait.Until(), wait.PollUntilContextTimeout()
│
├── apiserver/              Generic API server framework (used by kube-apiserver)
│   ├── pkg/endpoints/      HTTP handler routing
│   ├── pkg/authentication/ Auth plugins
│   ├── pkg/authorization/  Authz (RBAC)
│   ├── pkg/admission/      Admission framework
│   └── pkg/storage/        Storage interface + etcd3 implementation
│
├── client-go/              The official Go client library
│   ├── kubernetes/         Typed Clientset — client.CoreV1().Pods().Get()
│   ├── dynamic/            Dynamic client for CRDs / unknown types
│   ├── tools/cache/        Informers, Listers, WorkQueues ← most important
│   ├── tools/clientcmd/    Loads kubeconfig files
│   └── rest/               Low-level HTTP client
│
├── controller-manager/     Generic controller manager framework
│
├── kube-scheduler/         Public scheduler types and plugin interfaces
│   └── framework/          Plugin interface definitions
│
├── kubectl/                kubectl library
│   └── pkg/cmd/            Every subcommand implementation
│       ├── apply/apply.go  kubectl apply
│       ├── get/get.go      kubectl get
│       └── …
│
├── cri-api/                Container Runtime Interface (gRPC protobuf)
│
├── code-generator/         Tools to generate clientsets, informers, listers
│
└── sample-controller/      Canonical example of writing a Kubernetes controller
```

---

## api/ — OpenAPI Specs

```
api/
├── openapi-spec/           swagger.json and openapi.json (generated — do not edit)
└── api-rules/              Validation rules for API review (e.g. no new list types)
```

Not Go source. Consumed by docs generators and the API linter. Regenerated via `make update`.

---

## test/ — Test Code

```
test/
├── e2e/                    End-to-end tests (require a running cluster)
│   └── framework/          Test helpers used by e2e tests
├── e2e_node/               Node-level e2e (require a node with kubelet)
├── integration/            Integration tests (API server + etcd, no full cluster)
│   ├── scheduler/          Scheduler integration tests
│   ├── auth/               Auth integration tests
│   └── …
├── conformance/            Official conformance test suite
├── fixtures/               Test YAML manifests and test data
└── fuzz/                   Fuzz testing targets
```

**For contributors:** Integration tests are the most useful. They test real behavior without needing a full cluster. Run with `make test-integration WHAT=./test/integration/<area>/...`

---

## hack/ — Developer Scripts

The scripts you will use daily as a contributor:

| Script | When to use |
|--------|------------|
| `hack/local-up-cluster.sh` | Start a full local cluster for manual testing |
| `hack/update-codegen.sh` | After adding or changing API types (regenerates clients, informers, listers) |
| `hack/update-all.sh` | Run ALL generators — do this before submitting |
| `hack/verify-all.sh` | Run ALL verifiers — equivalent to what CI runs |
| `hack/verify-gofmt.sh` | Check formatting only |
| `hack/verify-golangci-lint.sh` | Run linter only |
| `hack/pin-dependency.sh` | Add or update a vendored dependency |
| `hack/update-vendor.sh` | Sync vendor/ after dependency changes |
| `hack/install-etcd.sh` | Install etcd locally (needed for local cluster) |
| `hack/boilerplate/boilerplate.go.txt` | License header template (required on every `.go` file) |

---

## Key File Patterns

### Pattern: Every binary is thin

```go
// cmd/kube-apiserver/apiserver.go — 37 lines total
func main() {
    command := app.NewAPIServerCommand()  // real work is in app/
    code := cli.Run(command)
    os.Exit(code)
}
```

Start reading from `cmd/<binary>/app/server.go`, never `main.go`.

### Pattern: Generated files — never hand-edit

```
zz_generated.deepcopy.go    ← generated by deepcopy-gen
zz_generated.defaults.go    ← generated by defaulter-gen
generated.pb.go             ← generated from .proto files
generated.proto             ← generated from Go types
```

If you edit these, your changes will be overwritten on the next `make update`. Fix the source, then run `make update`.

### Pattern: OWNERS files

Every directory has `OWNERS`:
```yaml
approvers:
  - sig-scheduling-approver
reviewers:
  - sig-scheduling-reviewer
```

Your PR must be approved by someone in `approvers` for the files you changed.

---

## How to Find Where Code Lives

| You want to find | Where to look |
|------------------|--------------|
| How Pod is stored in etcd | `pkg/registry/core/pod/storage/` |
| How kubectl apply works | `staging/src/k8s.io/kubectl/pkg/cmd/apply/apply.go` |
| How RBAC authorization works | `staging/src/k8s.io/apiserver/pkg/authorization/rbac/` |
| How a Deployment creates ReplicaSets | `pkg/controller/deployment/` |
| How the scheduler picks a node | `pkg/scheduler/schedule_one.go` |
| How kubelet starts a container | `pkg/kubelet/kuberuntime/kuberuntime_container.go` |
| Pod type definition | `staging/src/k8s.io/api/core/v1/types.go` |
| Informer implementation | `staging/src/k8s.io/client-go/tools/cache/` |
| Admission controller interface | `staging/src/k8s.io/apiserver/pkg/admission/interfaces.go` |

---

## Go Module Structure

This repo uses a **Go workspace** (`go.work`) to handle the multi-module layout:

```
go.work links:
  .                              (k8s.io/kubernetes)
  staging/src/k8s.io/api
  staging/src/k8s.io/apimachinery
  staging/src/k8s.io/client-go
  … (all 33 staging modules)
```

When you import `k8s.io/client-go/kubernetes`, Go resolves it to `staging/src/k8s.io/client-go/kubernetes` via the workspace — no network call needed.

**Never `go mod tidy`.** Dependency management uses:
```bash
# To add/update a dependency:
hack/pin-dependency.sh k8s.io/some-package v1.2.3
hack/update-vendor.sh
```
