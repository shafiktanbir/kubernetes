# ⭐ Core Concepts — Must Know Cold

---

## Q1: What is a Pod and why does K8s use it instead of running containers directly?

**Senior Answer:**

A Pod is the smallest deployable unit in Kubernetes — it's a wrapper around one or more containers that share the same network namespace and storage volumes.

The reason K8s doesn't run containers directly is that real-world applications often need tightly coupled helper processes. Classic example: your main app container alongside a logging sidecar that ships logs to Elasticsearch, or an Envoy proxy for service mesh. These need to share localhost networking and certain file paths — that's exactly what a Pod provides. They live and die together.

The practical implication: when K8s decides where to run your workload, it places the whole Pod on a node — never splits containers across nodes. So your sidecar is always co-located with your main app.

**Tradeoff to mention:** Because Pods are ephemeral by design, you never manage them directly in production — you use a Deployment or StatefulSet which manages Pod lifecycle for you.

---

## Q2: What's the difference between a Deployment and a StatefulSet? When do you use each?

**Senior Answer:**

A Deployment manages stateless Pods. All Pods are identical and interchangeable — K8s can kill any one of them and replace it with a new one, potentially on a different node, with a different IP. The Pod name changes after restart.

A StatefulSet manages stateful Pods. Each Pod gets:
- A stable, predictable name (`db-0`, `db-1`, `db-2`)
- A stable network identity (DNS: `db-0.db-service`)
- Its own PersistentVolumeClaim that follows it if it's rescheduled

**Use Deployment for:** web servers, API services, workers — anything that doesn't care about its identity.

**Use StatefulSet for:** databases (PostgreSQL, MySQL), message brokers (Kafka, RabbitMQ), anything with leader-election that needs to know "which replica am I?"

**The gotcha interviewers love:** When a node running a StatefulSet pod fails, K8s does NOT automatically reschedule those pods onto healthy nodes. It waits until the dead node comes back or you forcefully delete the pods. This is intentional — to prevent split-brain scenarios where you'd have two `db-0` pods running simultaneously.

---

## Q3: Explain the Kubernetes control plane components.

**Senior Answer:**

The control plane is the brain of the cluster. It has four main components:

**kube-apiserver** — the front door. Every interaction with the cluster goes through it: `kubectl`, ArgoCD, your CI/CD pipeline, the kubelets on nodes. It validates and stores state in etcd.

**etcd** — the cluster's source of truth. A distributed key-value store that holds the desired state of everything: deployments, pods, secrets, configmaps. If etcd is lost and you have no backup, your cluster config is gone. This is why production setups run 3 or 5 etcd instances.

**kube-scheduler** — watches for new Pods with no assigned node and selects the best node based on resource requests, affinity rules, taints/tolerations, and available capacity.

**kube-controller-manager** — runs a set of controllers in a loop. The Deployment controller watches: "Is the actual number of running pods equal to the desired replicas?" If not, it creates or deletes pods. Same logic for ReplicaSets, DaemonSets, etc.

**The architecture insight:** The control plane only manages desired state. Nodes are managed by the **kubelet** — an agent running on each worker node. Kubelet watches the API server for pods assigned to its node and ensures containers are running via the container runtime (containerd/CRI-O).

---

## Q4: What happens when you run `kubectl apply -f deployment.yaml`?

**Senior Answer:**

This is a great question to walk through the full request flow:

1. `kubectl` reads your kubeconfig, finds the API server address and your credentials
2. It sends an HTTP PATCH/POST request to `kube-apiserver`
3. The API server authenticates you (certificate or token), then authorizes via RBAC
4. It validates the manifest (schema check — is `replicas` a number? Does the image field exist?)
5. The desired state is written to **etcd**
6. The **Deployment controller** (inside kube-controller-manager) detects the new Deployment and creates a ReplicaSet
7. The ReplicaSet controller sees 0 pods exist, creates Pod objects in etcd
8. The **kube-scheduler** sees unscheduled Pods, evaluates nodes, and binds each Pod to a node (writes the node assignment to etcd)
9. The **kubelet** on the chosen node polls the API server, sees a Pod assigned to it, tells containerd to pull the image and start the container
10. Kubelet reports back: Pod is Running

The key insight: nothing is synchronous. `kubectl apply` returns as soon as the API server accepts the request. The actual container starting happens asynchronously. That's why you need `kubectl rollout status` to wait for completion.

---

## Q5: What is a Service and why do you need it if Pods already have IP addresses?

**Senior Answer:**

Pod IPs are ephemeral. When a Pod dies and a new one is created to replace it, it gets a completely different IP address. If your frontend is hardcoding `10.0.0.5` to talk to the backend, it breaks every time a backend pod restarts.

A Service provides a stable virtual IP (ClusterIP) and DNS name that never changes, regardless of which pods are behind it. K8s updates the endpoints automatically as pods come and go.

**The three Service types you need to know:**

- **ClusterIP** (default) — only reachable inside the cluster. Use for internal pod-to-pod communication.
- **NodePort** — exposes the service on a port on every node's IP. Rarely used in production — it's for development or bare-metal clusters without a cloud LB.
- **LoadBalancer** — provisions a cloud load balancer (AWS ELB, GCP LB). Use to expose services externally in cloud environments. Not relevant locally unless you have MetalLB.

**What about Ingress?** Services operate at L4 (TCP/UDP). Ingress operates at L7 (HTTP) and lets you route based on hostnames and paths. In production, you expose one LoadBalancer → Ingress controller, and all routing happens inside the cluster via Ingress rules.

---

## Q6: What's the difference between `kubectl apply` and `kubectl create`?

**Senior Answer:**

`kubectl create` is imperative — it creates a resource. Fails if the resource already exists.

`kubectl apply` is declarative — it compares your manifest against the current cluster state and applies the diff. If the resource doesn't exist, it creates it. If it exists, it patches only what changed. This is what you use in CI/CD and GitOps because it's idempotent — running it twice is safe.

In practice, you should **never use `kubectl create` in automation**. Use `kubectl apply`. The only exception is one-off imperative commands for quick debugging:
```bash
kubectl create namespace temp-test   # fine for quick things
```

But for anything that goes into a pipeline or gets committed to Git — always declarative YAML with `kubectl apply` or Helm.

---

## Q7: What is a Namespace and when should you use multiple namespaces?

**Senior Answer:**

A Namespace is a virtual cluster within a cluster. It provides:
- **Isolation** — resources in namespace A don't conflict with namespace B
- **Access control** — RBAC Roles can be scoped to a namespace
- **Resource quotas** — you can limit CPU/memory per namespace
- **Logical grouping** — team A's services are separate from team B's

**Common patterns:**

1. **Environment isolation:** `development`, `staging`, `production` — each environment in its own namespace. Simple, very common at smaller companies.

2. **Team isolation:** `team-payments`, `team-catalog`, `team-auth` — each team owns their namespace, has their own RBAC, their own resource quotas.

3. **Both combined:** `payments-production`, `payments-staging` — scales well for larger orgs.

**What namespaces do NOT isolate:**
- Network traffic (Pods can talk across namespaces by default — you need NetworkPolicy for that)
- Node resources (a pod in `development` can still consume all CPU on a node if no quotas are set)
- ClusterRoles (those are cluster-wide)

---

## Q8: What is a ConfigMap? What's the difference between ConfigMap and Secret?

**Senior Answer:**

A ConfigMap stores non-sensitive configuration as key-value pairs — environment variables, config file content, command-line arguments. It decouples configuration from your container image so the same image runs in dev, staging, and prod with different behavior.

A Secret is structurally identical but intended for sensitive data — passwords, API keys, certificates. The differences:

- Secrets are base64-encoded (not encrypted by default) but marked sensitive — K8s won't print them in logs
- RBAC can control access to Secrets separately from ConfigMaps
- Secrets can be encrypted at rest with KMS integration (AWS KMS, GCP KMS)
- Secrets can be memory-mounted (stored in tmpfs, never written to disk on the node)

**The honest caveat interviewers respect:** Base64 is not encryption. Anyone with access to etcd can decode Secrets. In high-security environments, companies use HashiCorp Vault or cloud-native secret managers (AWS Secrets Manager, GCP Secret Manager) with the CSI Secrets Store driver to inject secrets at pod startup — never storing them in etcd at all.

---

## Q9: What is a DaemonSet? Give a real use case.

**Senior Answer:**

A DaemonSet ensures exactly one Pod runs on every node in the cluster (or a subset of nodes matching a selector). When a new node joins the cluster, the DaemonSet automatically schedules a Pod on it. When a node is removed, the Pod is cleaned up.

**Real use cases:**
- **Log collection** — Fluentd or Filebeat running on every node, shipping logs from `/var/log` to Elasticsearch. Every node generates logs, so you need one collector per node.
- **Monitoring** — Prometheus node-exporter collecting host-level metrics (CPU, disk, network). Must run on every node to get per-node data.
- **Network plugins** — Calico, Cilium, Flannel run as DaemonSets because every node needs networking configured.
- **Security agents** — Falco for runtime threat detection runs on every node.

**The key distinction from Deployment:** A Deployment says "run N replicas somewhere." A DaemonSet says "run exactly one on each node." You can't have 2 Fluentd pods on one node and 0 on another — that defeats the purpose.

---

## Q10: What happens when a Pod's resource limit is exceeded?

**Senior Answer:**

It depends on which resource:

**CPU limit exceeded:** CPU is compressible. K8s throttles the container — it gets less CPU time, but it doesn't die. You'll see high latency and slow response times, but the Pod stays running. This is why CPU limits are controversial — some teams set requests but no limits for CPU, arguing throttling is worse than the occasional burst.

**Memory limit exceeded:** Memory is incompressible. The Linux kernel OOM killer terminates the container with exit code 137 (SIGKILL). K8s marks the pod as `OOMKilled` in `kubectl describe pod`. If the pod has a restart policy (which Deployments have by default), it restarts. If it keeps getting OOMKilled, you'll see `CrashLoopBackOff` with exponential backoff.

**Diagnosis command:**
```bash
kubectl describe pod <name>
# Look for: Last State: Terminated, Reason: OOMKilled
```

**Fix:** Either increase the memory limit or find the memory leak in your application. Common causes: memory leaks in the app code, loading too much data into memory, connection pools not being released.
