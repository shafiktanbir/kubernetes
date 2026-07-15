# Daily 2-Hour K8s Contributor Plan
> 30 sessions × 2 hours = 60 hours total.
> At the end: your first PR submitted to kubernetes/kubernetes.
>
> **Rules:**
> - Start a timer when you sit down. Stop at 2 hours. No exceptions.
> - Tick the checkbox only when you hit the "Done when" criteria exactly.
> - If a day's task takes less than 2 hours, use leftover time to re-read or run commands again.
> - Do not skip days. Do not do two days in one sitting.

---

## Progress Tracker

```
Week 1  [ ][ ][ ][ ][ ][ ][ ]   Go language
Week 2  [ ][ ][ ][ ][ ][ ][ ]   Go + environment + K8s concepts
Week 3  [ ][ ][ ][ ][ ][ ][ ]   K8s hands-on + codebase entry
Week 4  [ ][ ][ ][ ][ ][ ][ ]   Scheduler deep dive
Week 5+ [ ][ ][ ][ ][ ][ ][ ]   First contribution
```

---

## WEEK 1 — Go Language
> Goal: Read and write basic Go confidently before touching K8s code.
> You cannot understand K8s source without Go. Don't rush this week.

---

### Day 1 — Go Installation + Tour Basics
**Goal:** Install Go, run your first program, complete Tour basics.

**Tasks (2 hours):**
- [ ] Check Go version the repo needs: `cat /home/shafikul/Documents/coding/kubernetes/.go-version`
- [ ] Install that exact Go version from https://go.dev/dl/
- [ ] Verify: `go version`
- [ ] Open https://go.dev/tour/welcome/1 — complete sections:
  - Packages, Variables, Functions (15 slides)
  - Flow control: for, if, switch (14 slides)
- [ ] Write this program from scratch in a new file `~/go-practice/day01/main.go`:
  ```go
  package main

  import "fmt"

  func sum(nums []int) int {
      total := 0
      for _, n := range nums {
          total += n
      }
      return total
  }

  func main() {
      fmt.Println(sum([]int{1, 2, 3, 4, 5}))
  }
  ```
- [ ] Run it: `go run ~/go-practice/day01/main.go`

**Done when:** Program prints `15` without errors.

---

### Day 2 — Go Types, Structs, Interfaces
**Goal:** Understand structs and interfaces — K8s is built on them.

**Tasks (2 hours):**
- [ ] Continue Go Tour: "More types" section (pointers, structs, slices, maps)
- [ ] Continue Go Tour: "Methods and interfaces" section — read **all of it twice**
- [ ] Write this from scratch in `~/go-practice/day02/main.go`:
  ```go
  package main

  import "fmt"

  type Animal interface {
      Sound() string
      Name() string
  }

  type Dog struct{ name string }
  type Cat struct{ name string }

  func (d Dog) Sound() string { return "woof" }
  func (d Dog) Name() string  { return d.name }
  func (c Cat) Sound() string { return "meow" }
  func (c Cat) Name() string  { return c.name }

  func describe(a Animal) {
      fmt.Printf("%s says %s\n", a.Name(), a.Sound())
  }

  func main() {
      animals := []Animal{Dog{"Rex"}, Cat{"Whiskers"}}
      for _, a := range animals {
          describe(a)
      }
  }
  ```
- [ ] Run it. Understand why `describe()` doesn't need to know if it's a Dog or Cat.

**Done when:** You can explain in one sentence what an interface is and why K8s uses them everywhere.

---

### Day 3 — Goroutines and Channels
**Goal:** Understand concurrency. K8s runs dozens of goroutines per component.

**Tasks (2 hours):**
- [ ] Go Tour: "Concurrency" section — all slides
- [ ] Read this article: https://go.dev/blog/pipelines (30 min)
- [ ] Write `~/go-practice/day03/main.go`:
  ```go
  package main

  import (
      "fmt"
      "sync"
  )

  func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
      defer wg.Done()
      for j := range jobs {
          results <- j * j // square the number
      }
  }

  func main() {
      jobs := make(chan int, 10)
      results := make(chan int, 10)
      var wg sync.WaitGroup

      for w := 1; w <= 3; w++ {
          wg.Add(1)
          go worker(w, jobs, results, &wg)
      }

      for j := 1; j <= 9; j++ {
          jobs <- j
      }
      close(jobs)

      go func() {
          wg.Wait()
          close(results)
      }()

      for r := range results {
          fmt.Println(r)
      }
  }
  ```
- [ ] Run it. Observe that output order is non-deterministic.

**Done when:** You understand why results print in random order, and you can explain `chan`, `goroutine`, and `sync.WaitGroup` in plain English.

---

### Day 4 — Error Handling + context.Context
**Goal:** `context.Context` is in every K8s function signature. Must understand it.

**Tasks (2 hours):**
- [ ] Read: https://go.dev/blog/error-handling-and-go (20 min)
- [ ] Read: https://pkg.go.dev/context (read the package doc at the top, 20 min)
- [ ] Read: https://go.dev/blog/context (30 min)
- [ ] Write `~/go-practice/day04/main.go`:
  ```go
  package main

  import (
      "context"
      "fmt"
      "time"
  )

  func doWork(ctx context.Context, name string) error {
      select {
      case <-time.After(2 * time.Second):
          fmt.Printf("%s: work done\n", name)
          return nil
      case <-ctx.Done():
          return fmt.Errorf("%s: cancelled: %w", name, ctx.Err())
      }
  }

  func main() {
      ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
      defer cancel()

      if err := doWork(ctx, "task-1"); err != nil {
          fmt.Println("Error:", err)
      }
  }
  ```
- [ ] Run it. Notice the timeout fires before work completes.
- [ ] Change timeout to `3*time.Second`. Run again. Work completes.

**Done when:** You can explain why every K8s function takes `ctx context.Context` as its first argument.

---

### Day 5 — Go Modules + Reading Real Go Code
**Goal:** Understand Go modules, then read your first real K8s file.

**Tasks (2 hours):**
- [ ] Read: https://go.dev/blog/using-go-modules (20 min)
- [ ] Run: `cat /home/shafikul/Documents/coding/kubernetes/go.mod | head -30`
- [ ] Run: `cat /home/shafikul/Documents/coding/kubernetes/go.work`
- [ ] Open and read: `/home/shafikul/Documents/coding/kubernetes/staging/src/k8s.io/client-go/doc.go`
  - Read all 111 lines. This is production-quality Go documentation.
- [ ] Open and read: `/home/shafikul/Documents/coding/kubernetes/pkg/kubelet/doc.go`
- [ ] Open and read the first 50 lines of:
  `/home/shafikul/Documents/coding/kubernetes/cmd/kube-apiserver/apiserver.go`

**Done when:** You can explain what a Go workspace is and why K8s uses one instead of a single `go.mod`.

---

### Day 6 — Go Practice: Write a Mini Controller
**Goal:** Write the pattern that every K8s controller uses — before reading the real thing.

**Tasks (2 hours):**
- [ ] Write `~/go-practice/day06/main.go` — a simple reconcile loop:
  ```go
  package main

  import (
      "context"
      "fmt"
      "math/rand"
      "time"
  )

  type State struct {
      Desired int
      Actual  int
  }

  func observe(s *State) {
      // Simulate something randomly dying
      if rand.Intn(5) == 0 {
          s.Actual--
      }
  }

  func reconcile(ctx context.Context, s *State) {
      for {
          select {
          case <-ctx.Done():
              fmt.Println("controller stopped")
              return
          case <-time.After(1 * time.Second):
              observe(s)
              if s.Actual < s.Desired {
                  fmt.Printf("actual=%d desired=%d → creating 1\n", s.Actual, s.Desired)
                  s.Actual++
              } else {
                  fmt.Printf("actual=%d desired=%d → nothing to do\n", s.Actual, s.Desired)
              }
          }
      }
  }

  func main() {
      ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
      defer cancel()
      state := &State{Desired: 3, Actual: 3}
      reconcile(ctx, state)
  }
  ```
- [ ] Run it. Watch the reconciler keep `Actual` at 3.
- [ ] This is **exactly** what every Kubernetes controller does. Internalize it.

**Done when:** You feel comfortable writing Go with structs, goroutines, contexts, and channels.

---

### Day 7 — Week 1 Review + Go Tooling
**Goal:** Lock in Go knowledge, learn the tools you'll use every day.

**Tasks (2 hours):**
- [ ] Learn `gofmt`: run `gofmt -l ~/go-practice/` to find unformatted files
- [ ] Learn `go test`: write a test for your Day 6 `reconcile` function
  - File: `~/go-practice/day06/main_test.go`
  - Write one table-driven test that verifies reconcile closes the gap
- [ ] Learn `go vet`: run `go vet ./...` in your practice directory
- [ ] Read: https://go.dev/doc/effective_go — sections "Commentary", "Names", "Control structures" only (40 min)
- [ ] Answer these questions in a text file `~/go-practice/week1-notes.txt`:
  1. What is the difference between a pointer receiver and a value receiver?
  2. What happens if you send to a closed channel?
  3. Why does `defer` run even if a function panics?
  4. What is the zero value of a `map`? What happens if you write to it?

**Done when:** All 4 questions answered. You have at least one passing Go test.

---

## WEEK 2 — Environment + K8s Concepts + kubectl
> Goal: Set up the real environment. Understand K8s from the outside (user perspective) before reading code.

---

### Day 8 — Sign CLA + Fork + Clone + Build
**Goal:** Get the dev environment running. This day is all setup — expect friction.

**Tasks (2 hours):**
- [ ] **Sign the CLA first:** https://identity.linuxfoundation.org/projects/cncf
  - Use the same GitHub account you'll use for PRs
  - Takes 5 minutes. Do it now.
- [ ] Fork kubernetes/kubernetes on GitHub (your account)
- [ ] Your local clone is already at `/home/shafikul/Documents/coding/kubernetes`
- [ ] Add your fork as a remote:
  ```bash
  cd /home/shafikul/Documents/coding/kubernetes
  git remote add fork https://github.com/<your-username>/kubernetes.git
  git remote -v   # should show both upstream and fork
  ```
- [ ] Install etcd:
  ```bash
  hack/install-etcd.sh
  echo 'export PATH=$PATH:/home/shafikul/Documents/coding/kubernetes/third_party/etcd' >> ~/.bashrc
  source ~/.bashrc
  etcd --version
  ```
- [ ] Build the entire project:
  ```bash
  make
  ```
  This takes 5–15 minutes. While it builds, read `shafikul-guide/01-k8s-big-picture.md`.

**Done when:** `make` completes with no errors. `etcd --version` works.

---

### Day 9 — Install Kind + Run a Real Cluster + kubectl
**Goal:** Use Kubernetes as a user. You cannot contribute to something you've never used.

**Tasks (2 hours):**
- [ ] Install Kind: https://kind.sigs.k8s.io/docs/user/quick-start/#installation
  ```bash
  go install sigs.k8s.io/kind@latest
  ```
- [ ] Install kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- [ ] Create a cluster:
  ```bash
  kind create cluster --name k8s-study
  kubectl cluster-info
  ```
- [ ] Run these commands and understand each output:
  ```bash
  kubectl get nodes
  kubectl get pods -A             # all pods in all namespaces
  kubectl get namespaces
  kubectl describe node           # read the output top to bottom
  ```
- [ ] Create your first pod:
  ```bash
  kubectl run nginx --image=nginx
  kubectl get pods -w             # watch it go Pending → Running
  kubectl describe pod nginx      # read Events section carefully
  kubectl logs nginx
  kubectl delete pod nginx
  ```

**Done when:** You ran a pod and watched it go from Pending to Running. You read the Events section in `describe` output.

---

### Day 10 — Deployments, Services, ReplicaSets
**Goal:** Understand the controller chain: Deployment → ReplicaSet → Pod.

**Tasks (2 hours):**
- [ ] Create a Deployment:
  ```bash
  kubectl create deployment web --image=nginx --replicas=3
  kubectl get deployments
  kubectl get replicasets
  kubectl get pods
  ```
- [ ] Simulate a pod crash — delete one pod and watch it recreate:
  ```bash
  kubectl delete pod <one-of-the-pod-names>
  kubectl get pods -w
  ```
  **Why did it recreate?** — Write the answer in your notes.
- [ ] Expose it as a Service:
  ```bash
  kubectl expose deployment web --port=80 --type=ClusterIP
  kubectl get service web
  kubectl describe service web    # look at Endpoints
  ```
- [ ] Scale it:
  ```bash
  kubectl scale deployment web --replicas=5
  kubectl get pods
  kubectl scale deployment web --replicas=1
  kubectl get pods -w
  ```
- [ ] Delete everything:
  ```bash
  kubectl delete deployment web
  kubectl delete service web
  ```
- [ ] Read official doc: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

**Done when:** You can explain why deleting one pod in a Deployment doesn't reduce the count permanently.

---

### Day 11 — ConfigMaps, Secrets, Namespaces + RBAC Basics
**Goal:** Understand config injection and access control — they appear everywhere in the codebase.

**Tasks (2 hours):**
- [ ] Create a ConfigMap and use it in a pod:
  ```bash
  kubectl create configmap app-config --from-literal=LOG_LEVEL=debug --from-literal=PORT=8080
  kubectl describe configmap app-config
  ```
  Write a pod YAML that mounts this ConfigMap as env vars and apply it.
- [ ] Create a Secret:
  ```bash
  kubectl create secret generic db-creds --from-literal=password=supersecret
  kubectl get secret db-creds -o yaml
  ```
  Notice: the value is base64 encoded, **not encrypted**. Decode it:
  ```bash
  echo "c3VwZXJzZWNyZXQ=" | base64 -d
  ```
- [ ] Create a Namespace and deploy into it:
  ```bash
  kubectl create namespace study
  kubectl run test-pod --image=nginx -n study
  kubectl get pods -n study
  kubectl delete namespace study   # deletes everything inside it
  ```
- [ ] Read: https://kubernetes.io/docs/reference/access-authn-authz/rbac/ (concepts section, 30 min)

**Done when:** You understand why Secrets are not secure by default and what "encryption at rest" means.

---

### Day 12 — Read shafikul-guide Docs
**Goal:** Consolidate architecture knowledge before touching source code.

**Tasks (2 hours):**
- [ ] Read `shafikul-guide/01-k8s-big-picture.md` — all of it, slowly. Take notes.
- [ ] Read `shafikul-guide/02-codebase-map.md` — all of it.
- [ ] Open the actual directories while reading — verify what the doc says:
  ```bash
  ls /home/shafikul/Documents/coding/kubernetes/cmd/
  ls /home/shafikul/Documents/coding/kubernetes/pkg/
  ls /home/shafikul/Documents/coding/kubernetes/staging/src/k8s.io/
  ```
- [ ] Write in your notes: the answer to these questions from memory (don't look):
  1. Why does `cmd/kube-apiserver/apiserver.go` have only 5 lines?
  2. What is the difference between `pkg/` and `staging/`?
  3. What does `api/` contain? (Not Go source — what is it?)
  4. Name 3 scripts in `hack/` and what they do.

**Done when:** All 4 questions answered from memory.

---

### Day 13 — Request Flow Trace (Read the doc + open the files)
**Goal:** Trace `kubectl apply` with the actual files open.

**Tasks (2 hours):**
- [ ] Open `shafikul-guide/03-request-flow.md`
- [ ] For each step in the request flow, open the actual file mentioned:
  ```bash
  # Step 1 — kubectl apply
  cat staging/src/k8s.io/kubectl/pkg/cmd/apply/apply.go | head -80

  # Step 2 — API server handler
  ls staging/src/k8s.io/apiserver/pkg/endpoints/handlers/

  # Step 8 — Scheduler Run()
  grep -n "func.*Run" pkg/scheduler/scheduler.go

  # Step 10 — Kubelet syncLoop
  grep -n "func.*syncLoop" pkg/kubelet/kubelet.go
  ```
- [ ] Draw the full flow on paper (or a text file). From `kubectl` to `container running`. No looking.
- [ ] Keep your drawing. You'll add to it as you learn more.

**Done when:** You have a hand-drawn (or text) diagram of the full request flow.

---

### Day 14 — Week 2 Review + Kind Cleanup
**Goal:** Consolidate week 2. Make sure the environment is solid.

**Tasks (2 hours):**
- [ ] Run unit tests for the scheduler to confirm build is healthy:
  ```bash
  cd /home/shafikul/Documents/coding/kubernetes
  make test WHAT=./pkg/scheduler/... 2>&1 | tail -30
  ```
- [ ] Answer from memory:
  1. Name every component in the K8s control plane and one-sentence description of each.
  2. What does `kube-proxy` do? What does it NOT do?
  3. What is a `Watch` in the K8s API? Why is it better than polling?
  4. What is the difference between a `Deployment` and a `ReplicaSet`?
- [ ] Clean up Kind cluster:
  ```bash
  kind delete cluster --name k8s-study
  ```
- [ ] Join Kubernetes Slack: https://slack.k8s.io
  - Join `#kubernetes-contributors` and `#sig-scheduling`
  - Read the last 50 messages in `#sig-scheduling`. What are they discussing?

**Done when:** All 4 questions answered. You are in Kubernetes Slack.

---

## WEEK 3 — Scheduler Codebase Deep Dive
> Goal: Understand the scheduler well enough to find and fix a real issue.
> You chose the scheduler. Stick with it. Do not read kubelet or API server code this week.

---

### Day 15 — Scheduler Entry Point
**Goal:** Trace the scheduler from binary to first meaningful function.

**Tasks (2 hours):**
- [ ] Read `cmd/kube-scheduler/scheduler.go` — 5 lines, understand each line
- [ ] Read `cmd/kube-scheduler/app/server.go` — focus on:
  - What flags does it accept?
  - What function does it call to start the scheduler?
- [ ] Find `NewSchedulerCommand` and trace it to `Scheduler.Run()`:
  ```bash
  grep -n "NewSchedulerCommand\|func Run\|func.*Run(" cmd/kube-scheduler/app/server.go | head -20
  ```
- [ ] Read `pkg/scheduler/scheduler.go` lines 1–122:
  - The `Scheduler` struct — what fields does it have? What does each one do?
  - The `Run()` function (line 525) — what does it start?
- [ ] Write in your notes: what are the 3 goroutines/loops that the scheduler starts in `Run()`?

**Done when:** You can describe what happens in the first 500ms after `kube-scheduler` binary starts.

---

### Day 16 — The Scheduling Queue
**Goal:** Understand how pods wait to be scheduled. Queue = the scheduler's inbox.

**Tasks (2 hours):**
- [ ] Read: `pkg/scheduler/backend/queue/` — list all files:
  ```bash
  ls pkg/scheduler/backend/queue/
  ```
- [ ] Read `pkg/scheduler/backend/queue/scheduling_queue.go` — first 100 lines only.
  Focus on:
  - What are the 3 sub-queues? (activeQ, backoffQ, unschedulablePods)
  - What is the difference between them?
- [ ] Read `pkg/scheduler/eventhandlers.go` — first 60 lines.
  - What events cause a pod to be added to the queue?
- [ ] Answer in notes:
  1. What is a "backoff" queue and why does it exist?
  2. When a node is added to the cluster, what happens to pods in `unschedulablePods`?
  3. Why does the scheduler not call the API server directly to get pods — it uses an informer instead. What is the benefit?

**Done when:** You understand why there are 3 queues, not 1.

---

### Day 17 — ScheduleOne: The Core Loop
**Goal:** Read the main scheduling function — the heart of the scheduler.

**Tasks (2 hours):**
- [ ] Open `pkg/scheduler/schedule_one.go`
- [ ] Read the first 100 lines carefully. Find `ScheduleOne` function.
- [ ] Trace the call chain inside `ScheduleOne`:
  ```bash
  grep -n "func \|\.Run\|PreFilter\|Filter\|Score\|Bind" pkg/scheduler/schedule_one.go | head -40
  ```
- [ ] Find where these plugin phases are called:
  - PreFilter
  - Filter  
  - Score
  - Bind
  Note the line number of each in your notes.
- [ ] Read `pkg/scheduler/framework/interface.go` lines 186–332 (the `Framework` interface).
  - Count how many plugin extension points exist.
  - Pick 3 and write what they do.

**Done when:** You have a flow diagram of `ScheduleOne()` with the 6 plugin phases and their sequence.

---

### Day 18 — Filter Plugins: Read a Real Plugin
**Goal:** Read a complete, real scheduler plugin from top to bottom.

**Tasks (2 hours):**
- [ ] List all filter plugins:
  ```bash
  ls pkg/scheduler/framework/plugins/
  ```
- [ ] Read `pkg/scheduler/framework/plugins/noderesources/fit.go` — **all of it**.
  This is the plugin that checks if a node has enough CPU/memory for a pod.
  - What is the plugin's `Name()`?
  - What does `Filter()` actually check?
  - What does `PreFilter()` compute and store in `CycleState`?
  - What does the `InsufficientResource` struct hold?
- [ ] Read the test file: `pkg/scheduler/framework/plugins/noderesources/fit_test.go` — first 100 lines.
  - What test cases does it cover?
- [ ] Run the tests for this plugin:
  ```bash
  make test WHAT=./pkg/scheduler/framework/plugins/noderesources/... GOFLAGS=-v
  ```

**Done when:** You can explain what `NodeResourcesFit` does without looking at the code.

---

### Day 19 — Score Plugins + the Cache
**Goal:** Understand scoring and the local cache that avoids hitting the API server.

**Tasks (2 hours):**
- [ ] Read `pkg/scheduler/framework/plugins/noderesources/least_allocated.go`
  - What does "least allocated" mean as a scoring strategy?
  - How does it calculate the score?
- [ ] Read `pkg/scheduler/backend/cache/` — list files:
  ```bash
  ls pkg/scheduler/backend/cache/
  ```
- [ ] Read `pkg/scheduler/backend/cache/cache.go` — first 80 lines.
  - What data does the scheduler cache store?
  - Why does the scheduler need its own cache instead of calling the API server on every scheduling decision?
- [ ] Run the cache tests:
  ```bash
  make test WHAT=./pkg/scheduler/backend/cache/... GOFLAGS=-v 2>&1 | tail -20
  ```
- [ ] Write in notes: if you have 5000 nodes and 100 pods/sec scheduling rate, how many API calls would be needed WITHOUT the cache? What happens WITH it?

**Done when:** You understand why a local cache is not optional — it's a performance requirement.

---

### Day 20 — Bind + Event Handlers + Informers
**Goal:** Understand how a scheduling decision gets written back, and how events re-trigger scheduling.

**Tasks (2 hours):**
- [ ] Find the bind step in `pkg/scheduler/schedule_one.go`:
  ```bash
  grep -n "Bind\|bind\|nodeName" pkg/scheduler/schedule_one.go
  ```
- [ ] Read the default bind plugin: `pkg/scheduler/framework/plugins/defaultbinder/default_binder.go`
  - What API call does it actually make to assign a pod to a node?
- [ ] Read `pkg/scheduler/eventhandlers.go` — focus on:
  - `addNodeToCache` — what triggers this?
  - `updatePodInQueue` — what triggers this?
  - Why does adding a new Node potentially unblock pods stuck in `unschedulablePods`?
- [ ] Understand the Informer pattern by reading:
  `staging/src/k8s.io/client-go/tools/cache/` — list files, then read `store.go` first 50 lines.

**Done when:** You can explain the complete cycle: pod created → queued → scheduled → nodeName written → kubelet picks it up.

---

### Day 21 — Week 3 Review: Write a Scheduler Summary
**Goal:** Verify you actually understand the scheduler. Writing reveals gaps.

**Tasks (2 hours):**
- [ ] Write a document `~/scheduler-notes.md` with these sections:
  1. **What the scheduler does** (2–3 sentences, no jargon)
  2. **The scheduling queue** — 3 sub-queues, what goes in each, what moves pods between them
  3. **The scheduling pipeline** — list all plugin phases in order with one sentence each
  4. **The cache** — what it stores, why it exists
  5. **How a result gets committed** — what function, what API call
  6. **What I still don't understand** — be honest, list them
- [ ] For every gap in section 6, find the file in the repo that would answer it.
  Note the file path next to each gap.

**Done when:** `~/scheduler-notes.md` exists and has content in all 6 sections.

---

## WEEK 4 — Find an Issue and Fix It
> Goal: Go from "I understand the scheduler" to "I have code ready to submit."

---

### Day 22 — Find Your Issue
**Goal:** Identify a specific, real, unassigned issue you can fix.

**Tasks (2 hours):**
- [ ] Search for scheduler good-first issues:
  ```
  https://github.com/kubernetes/kubernetes/issues?q=label%3Aarea%2Fscheduler+label%3A%22good+first+issue%22+is%3Aopen
  ```
- [ ] If no scheduler issues: search kubectl:
  ```
  https://github.com/kubernetes/kubernetes/issues?q=label%3Aarea%2Fkubectl+label%3A%22good+first+issue%22+is%3Aopen
  ```
- [ ] Read **every** open issue in the results. Not just titles — read the full issue body.
- [ ] For each issue you're interested in:
  - Is it already assigned to someone? (Check comments)
  - Is there already a linked PR?
  - Do you understand what needs to change?
  - Can you find the relevant file in the repo?
- [ ] Pick exactly one issue. Comment on it:
  > "I'd like to work on this. I've been reading through [specific file]. Is this still unassigned?"
- [ ] While waiting for confirmation, read that file completely.

**Done when:** You have commented on a specific issue and identified the exact file to change.

---

### Day 23 — Understand the Issue Deeply
**Goal:** Know exactly what to change before writing a single line.

**Tasks (2 hours):**
- [ ] Create your branch:
  ```bash
  cd /home/shafikul/Documents/coding/kubernetes
  git fetch upstream
  git checkout -b fix/<short-description> upstream/main
  ```
- [ ] Find every file related to the issue:
  ```bash
  grep -rn "the specific function or string" --include="*.go" pkg/scheduler/
  ```
- [ ] Read those files completely. Not just the area around the bug — the whole file.
- [ ] Find the existing tests for the code you'll change:
  ```bash
  ls pkg/scheduler/*_test.go
  ```
- [ ] Run existing tests — confirm they pass before you change anything:
  ```bash
  make test WHAT=./pkg/scheduler/... 2>&1 | tail -10
  ```
- [ ] Write in notes: exactly what change you need to make (specific function, specific line, specific new behavior).

**Done when:** You can describe the exact change in one paragraph before writing any code.

---

### Day 24 — Write the Fix
**Goal:** Make the actual code change.

**Tasks (2 hours):**
- [ ] Make your change. Keep it minimal — touch only what the issue requires.
- [ ] Follow the existing code style exactly:
  - Same indentation
  - Same error message format
  - Same log format (`klog.V(n).InfoS(...)`)
  - Same comment style
- [ ] Add or update tests in the corresponding `_test.go` file.
  - At minimum: one test case that would have caught the bug.
  - Follow the existing table-driven test pattern.
- [ ] Run the tests:
  ```bash
  make test WHAT=./pkg/scheduler/... GOFLAGS=-v
  ```
- [ ] If tests fail: fix them. Do not move on with failing tests.

**Done when:** All existing tests pass. Your new test passes. The bug behavior is covered.

---

### Day 25 — Verify + Write Commit Message
**Goal:** Make the PR ready to submit.

**Tasks (2 hours):**
- [ ] Check the license header on any new `.go` files you created.
  Copy from: `hack/boilerplate/boilerplate.go.txt`
- [ ] Format your code:
  ```bash
  gofmt -w <your-changed-files>
  ```
- [ ] Run all verifiers:
  ```bash
  make verify 2>&1 | tail -30
  ```
  Fix any failures. Common ones:
  - `gofmt` — run `gofmt -w` on the file
  - `imports` — run `goimports -w` on the file  
  - `lint` — read the specific message and fix it
- [ ] Stage your changes carefully:
  ```bash
  git diff          # review every line of your change
  git add -p        # stage interactively, not git add .
  ```
- [ ] Write your commit message (follow the format from `05-contribution-workflow.md`):
  ```
  scheduler: <what changed>
  
  <why it was wrong>
  <what you changed and why>
  
  Fixes #<issue-number>
  ```
- [ ] Commit:
  ```bash
  git commit
  ```

**Done when:** `make verify` passes. Commit is clean. Message follows the format.

---

### Day 26 — Submit the PR
**Goal:** Open the PR and post in Slack.

**Tasks (2 hours):**
- [ ] Push to your fork:
  ```bash
  git push fork fix/<your-branch-name>
  ```
- [ ] Open the PR on GitHub:
  - Go to your fork → "Compare & pull request"
  - Title: same as commit message first line
  - Body: use template from `05-contribution-workflow.md`
- [ ] Wait for CI to start (5–10 min). Watch the status.
- [ ] Post in `#sig-scheduling` on Slack:
  ```
  Hi folks — I've opened a small PR for issue #XXXXX.
  [link to PR]
  Would appreciate a review when someone has time.
  ```
- [ ] While waiting: re-read `shafikul-guide/03-request-flow.md` and update your hand-drawn diagram with anything you learned this week.

**Done when:** PR is open. CI is green. Slack message posted.

---

### Day 27 — Address CI + Start Looking for Issue #2
**Goal:** Handle CI failures if any. Start preparing for your second contribution.

**Tasks (2 hours):**
- [ ] Check CI status on your PR. Fix any failures:
  - If `verify-gofmt` fails: `gofmt -w <file>`, push new commit
  - If `verify-imports` fails: `goimports -w <file>`, push new commit
  - If unit test fails: read the failure, fix your test
- [ ] While waiting for review, search for your next issue:
  ```
  https://github.com/kubernetes/kubernetes/issues?q=label%3Aarea%2Fscheduler+label%3A%22help+wanted%22+is%3Aopen
  ```
- [ ] Join the SIG Scheduling mailing list:
  `https://groups.google.com/a/kubernetes.io/g/sig-scheduling`
- [ ] Find the next SIG Scheduling meeting on the calendar and mark it.
  Plan to attend as a listener.

**Done when:** CI is green on your PR. You have a candidate second issue bookmarked.

---

### Day 28 — Respond to Review (or Review Others)
**Goal:** Participate in the review process — giving or receiving.

**Tasks (2 hours):**
- [ ] If review comments arrived on your PR: address every single one.
  - For each comment: read, understand, implement, respond
  - Push fixes as a new commit (do not squash yet)
  - Post summary comment on the PR
- [ ] If no review yet: review someone else's PR in the same area.
  - Search: `https://github.com/kubernetes/kubernetes/pulls?q=label%3Aarea%2Fscheduler+is%3Aopen`
  - Read one PR completely. Add a constructive comment if you spot something.
  - This is how you build reputation before you have a merged PR.
- [ ] Read one of the foundational KEPs:
  `https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/624-scheduling-framework`
  This explains WHY the scheduler framework is designed the way it is.

**Done when:** Either PR feedback addressed, or you left a thoughtful comment on someone else's PR.

---

### Day 29 — Deep Dive: One Scheduler Plugin Fully
**Goal:** Pick one plugin you haven't read yet and understand it completely.

**Tasks (2 hours):**
- [ ] Pick one plugin from: `ls pkg/scheduler/framework/plugins/`
  Good choices: `nodeaffinity`, `podtopologyspread`, `tainttoleration`
- [ ] Read the entire plugin `.go` file
- [ ] Read the entire `_test.go` file
- [ ] Run its tests:
  ```bash
  make test WHAT=./pkg/scheduler/framework/plugins/<plugin-name>/... GOFLAGS=-v
  ```
- [ ] Write in notes:
  - What scheduling constraint does this plugin enforce?
  - Which extension points does it implement (Filter? Score? Both?)
  - What does it store in `CycleState` during PreFilter?
  - What is the most interesting test case?

**Done when:** You can explain this plugin as clearly as you explained `NodeResourcesFit`.

---

### Day 30 — Reflect + Plan Next 30 Days
**Goal:** Assess where you are. Plan the next phase.

**Tasks (2 hours):**
- [ ] Write a self-assessment in `~/progress-day30.md`:
  1. What do I understand well? (be specific — functions, packages)
  2. What is still fuzzy?
  3. What did the PR process feel like?
  4. Do I want to stay in SIG Scheduling or switch to another SIG?
- [ ] Check the status of your PR. If merged: 🎉 you're a Kubernetes contributor.
- [ ] Look at your `~/scheduler-notes.md` from Day 21. Update it with what you know now.
- [ ] Find your second issue and comment on it.
- [ ] Read the SIG Scheduling meeting notes from the last meeting:
  `https://github.com/kubernetes/community/tree/master/sig-scheduling`
  What are they working on for the next release? Write it in your notes.

**Done when:** Self-assessment written. Second issue identified and commented.

---

## After Day 30 — What Comes Next

By now you have:
- 1 PR merged (or in review)
- Deep knowledge of the scheduler package
- A presence in Slack and on GitHub
- Enough context to read SIG meeting notes and understand them

**Your next 30 days:**
- Fix 3–5 more issues (aim for 1 merged PR every 2 weeks)
- Attend one SIG Scheduling video call
- Start reading the kubelet or API server (your second subsystem)
- Apply for **Kubernetes Membership** after your 5th merged PR

**Membership requirements:**
- 5+ contributions merged
- 2 existing members sponsor you
- 3 months of active contribution
- Apply: https://github.com/kubernetes/community/blob/master/community-membership.md

---

## Daily Log (Optional but Useful)

Use this section to track what you actually did each day.

```
Day 01: _______________________________________________
Day 02: _______________________________________________
Day 03: _______________________________________________
Day 04: _______________________________________________
Day 05: _______________________________________________
Day 06: _______________________________________________
Day 07: _______________________________________________
Day 08: _______________________________________________
Day 09: _______________________________________________
Day 10: _______________________________________________
Day 11: _______________________________________________
Day 12: _______________________________________________
Day 13: _______________________________________________
Day 14: _______________________________________________
Day 15: _______________________________________________
Day 16: _______________________________________________
Day 17: _______________________________________________
Day 18: _______________________________________________
Day 19: _______________________________________________
Day 20: _______________________________________________
Day 21: _______________________________________________
Day 22: _______________________________________________
Day 23: _______________________________________________
Day 24: _______________________________________________
Day 25: _______________________________________________
Day 26: _______________________________________________
Day 27: _______________________________________________
Day 28: _______________________________________________
Day 29: _______________________________________________
Day 30: _______________________________________________
```
