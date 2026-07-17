# 🚀 Kubernetes with k3d — Scenario-Based Learning Guide

> **Your machine:** Core i3, 12GB RAM — perfectly fine for all labs here.
> **Approach:** Every lab gives you a **real problem**. You figure it out. Then the solution is below.

---

## Prerequisites (Do This Once)

Before you can run any Kubernetes labs locally, you need three tools installed on your machine:
**Docker** (the container runtime), **k3d** (a tool that runs a lightweight Kubernetes cluster inside Docker containers), and **kubectl** (the command-line tool you use to talk to any Kubernetes cluster).

---

### Step 1 — Install Docker

k3d runs Kubernetes nodes as Docker containers, so Docker **must** be installed and running first.
The official `get.docker.com` script installs `docker-ce` (Docker Community Edition) directly from Docker's own repository — this is always up-to-date, unlike the `docker.io` package available in Ubuntu's default repos which is often months out of date.

After installing, the `usermod` command adds your current user to the `docker` group so you can run Docker commands **without `sudo`**. `newgrp docker` activates that group change immediately without requiring a full logout.

```bash
# Download and run the official Docker install script
curl -fsSL https://get.docker.com | sudo sh

# Add your user to the 'docker' group — avoids needing sudo for every docker command
sudo usermod -aG docker $USER

# Activate the group change in the current shell session (no logout needed)
newgrp docker
```

> **Verify:** Run `docker run hello-world` — you should see a success message with no permission errors.

---

### Step 2 — Install k3d

k3d is a lightweight wrapper that runs [k3s](https://k3s.io) (a minimal Kubernetes distribution) inside Docker containers. It lets you spin up a full multi-node Kubernetes cluster on your laptop in seconds.

The install script fetches the latest k3d binary from GitHub and places it in `/usr/local/bin`.

```bash
# Download and run the official k3d install script
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

> **Verify:** Run `k3d version` — you should see the installed version printed.

---

### Step 3 — Install kubectl

`kubectl` is the official Kubernetes CLI. Every command you run to deploy apps, inspect pods, or debug issues goes through `kubectl`. We install it via the official Kubernetes APT repository (`pkgs.k8s.io`) — this is the current recommended method from the Kubernetes project (the older Google-hosted repo was deprecated in 2023).

Here's what each step does:

- **`apt-get install apt-transport-https ca-certificates curl gnupg`** — installs dependencies needed to add a secure HTTPS-based APT repository.
- **`mkdir -p -m 755 /etc/apt/keyrings`** — creates a directory to store the GPG signing key for the Kubernetes repo.
- **`curl ... | sudo gpg --dearmor`** — downloads the Kubernetes package signing key and converts it to a binary format APT can use to verify package authenticity.
- **`echo 'deb ...' | sudo tee`** — adds the official Kubernetes APT repository to your sources list.
- **`apt-get install kubectl`** — installs the actual `kubectl` binary.

```bash
# Refresh package lists
sudo apt-get update

# Install dependencies needed to add a new HTTPS apt repository securely
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# Create the directory that will store the GPG signing key
sudo mkdir -p -m 755 /etc/apt/keyrings

# Download the Kubernetes package signing key and save it in binary (dearmored) format
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Register the official Kubernetes apt repository as a trusted package source
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Refresh package lists again now that the new repo is registered
sudo apt-get update

# Install kubectl
sudo apt-get install -y kubectl
```

> **Verify:** Run `kubectl version --client` — you should see version info printed (no cluster connection needed).

---

### Step 4 — Create Your Practice Cluster

> **❓ Do I need to install k3s separately?**
> **No.** You only install **Docker**, **k3d**, and **kubectl** manually. k3d automatically pulls k3s as a Docker image (`rancher/k3s`) when you create a cluster — it runs k3s *inside* the Docker containers it manages. You never touch k3s directly.
>
> The dependency chain is: `Docker → k3d → k3s (auto-pulled) → your cluster`

Now that all the tools are in place, this single command creates the actual Kubernetes cluster. k3d spins up Docker containers to act as cluster nodes, and pulls the k3s image automatically on first run (may take a minute).

- **`practice`** — the name for this cluster (used in all lab exercises). You can name it anything; k3d will prefix container names with `k3d-practice-`.
- **`--servers 1`** — creates 1 control-plane node (the "brain" of the cluster: runs the API server, scheduler, and etcd).
- **`--agents 2`** — creates 2 worker nodes (where your actual app pods run). Total: 3 nodes.
- **`--port "80:80@loadbalancer"`** — maps port 80 on your laptop to port 80 on the k3d load balancer container. This is how HTTP traffic from your browser reaches services inside the cluster.
- **`--port "443:443@loadbalancer"`** — same thing for HTTPS traffic on port 443.

k3d also automatically updates your `~/.kube/config` file so `kubectl` points to this new cluster immediately — no manual context switching needed.

```bash
# This single command: pulls k3s, creates 3 nodes, configures kubeconfig
k3d cluster create practice \
  --servers 1 \
  --agents 2 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"
```

---

### Step 5 — Verify the Cluster

```bash
# List all nodes in the cluster with their status and roles
kubectl get nodes

# Expected output — 3 nodes total:
# NAME                     STATUS   ROLES                  AGE   VERSION
# k3d-practice-server-0    Ready    control-plane,master   30s   v1.31.x
# k3d-practice-agent-0     Ready    <none>                 28s   v1.31.x
# k3d-practice-agent-1     Ready    <none>                 28s   v1.31.x
```

---

## Lab Structure

| Level | Lab | Scenario |
|-------|-----|---------|
| 🟢 Beginner | [Lab 01](./beginner/lab-01-first-pod.md) | Your first deployment — something is broken, fix it |
| 🟢 Beginner | [Lab 02](./beginner/lab-02-scaling.md) | Black Friday — traffic spike, scale NOW |
| 🟢 Beginner | [Lab 03](./beginner/lab-03-rolling-update.md) | Deploy new version without downtime |
| 🟡 Intermediate | [Lab 04](./intermediate/lab-04-configmaps.md) | App behaves differently in dev vs prod |
| 🟡 Intermediate | [Lab 05](./intermediate/lab-05-secrets.md) | DB password must not be in Git |
| 🟡 Intermediate | [Lab 06](./intermediate/lab-06-ingress.md) | Expose your app to the outside world |
| 🟡 Intermediate | [Lab 07](./intermediate/lab-07-rbac.md) | New intern shouldn't touch production |
| 🔴 Advanced | [Lab 08](./advanced/lab-08-hpa.md) | App auto-scales based on CPU load |
| 🔴 Advanced | [Lab 09](./advanced/lab-09-node-failure.md) | A node dies — what happens? |
| 🔴 Advanced | [Lab 10](./advanced/lab-10-argocd.md) | Deploy with ArgoCD — GitOps style |
| 🔴 Advanced | [Lab 11](./advanced/lab-11-helm.md) | Package your app as a Helm chart |
| 🔴 Advanced | [Lab 12](./advanced/lab-12-monitoring.md) | App is slow — find out why using Prometheus |

---

## How to Use This Guide

1. **Read the scenario** — understand the real-world problem
2. **Try to solve it yourself** — use `kubectl`, write YAML
3. **Stuck?** Read the hints section
4. **Still stuck?** Read the full solution
5. **Verify** your solution matches the expected outcome
6. **Break it** on purpose — learn what happens

---

## Useful Commands Cheatsheet

A quick reference for commands you'll use constantly across all labs.

### Cluster Management

These commands control the lifecycle of your k3d cluster. Use `stop`/`start` to pause and resume between sessions (saves memory). Use `delete` to completely tear it down and start fresh.

```bash
k3d cluster list              # show all k3d clusters on this machine
k3d cluster stop practice     # pause the cluster (containers stopped, data preserved)
k3d cluster start practice    # resume the cluster after a stop
k3d cluster delete practice   # permanently destroy the cluster and all its data
```

---

### Node Management

These commands let you simulate node-level failures — useful for testing how Kubernetes reschedules pods when infrastructure goes down.

```bash
k3d node list                            # list all k3d-managed nodes (containers)
k3d node stop k3d-practice-agent-0       # simulate a node going offline (stops the container)
k3d node start k3d-practice-agent-0      # bring the node back online
```

> **What to watch:** After stopping a node, run `kubectl get pods -A` and observe Kubernetes detecting the failure and rescheduling pods to the remaining healthy node.

---

### Debugging Pods

These are the commands you'll reach for first whenever something is wrong. The typical debugging flow is: check pod status → describe for events → check logs → exec in for live inspection.

```bash
# List all pods across every namespace — gives you a full picture of the cluster state
kubectl get pods -A

# Show detailed info about a pod: its events, volumes, image, restart count, IP, node placement
# This is your first stop when a pod is stuck in Pending/CrashLoopBackOff/Error
kubectl describe pod <name>

# Print the application's stdout/stderr logs — what your app itself is printing
kubectl logs <pod-name>

# Print logs from the PREVIOUS container instance — critical when a pod keeps crashing and restarting
# (the current container starts fresh; --previous gives you the crash logs)
kubectl logs <pod-name> --previous

# Open an interactive shell INSIDE the running container — lets you inspect files,
# test network connectivity (curl, nslookup), and run commands in the exact app environment
kubectl exec -it <pod-name> -- bash

# List all cluster events sorted by time — shows scheduling decisions, image pull failures,
# OOM kills, probe failures, etc. Uses creationTimestamp for reliable sorting across all event types.
kubectl get events --sort-by='.metadata.creationTimestamp'
```

---

### Quick Cleanup

```bash
# Delete all resources (pods, services, deployments, etc.) in the current namespace.
# Useful for resetting between lab exercises. Does NOT delete PersistentVolumes or namespaces.
kubectl delete all --all
```

---

## 🎓 After the Labs — Move to k3s (Real Experience)

Once you finish all 12 labs here, you know how Kubernetes works. The next step is getting **real production experience** by running k3s directly on actual infrastructure — no Docker wrapping, no simulation.

> **Why k3s and not full Kubernetes?**
> k3s is production-grade, used by real companies, and takes 30 seconds to install. Full Kubernetes (kubeadm) requires significantly more setup. For small infra (a VPS, a Raspberry Pi cluster, or a home server), k3s is the right tool — not a compromise.

---

### What you need

| Option | Cost | Spec | Good for |
|---|---|---|---|
| **VPS** (Hetzner, DigitalOcean, Linode) | ~$5–10/month | 2 vCPU, 4GB RAM | Best starting point |
| **Old laptop / PC** | Free | Any x86_64 | Home lab |
| **Raspberry Pi 4** | ~$50 one-time | 4GB RAM model | Edge / ARM experience |
| **Two VMs locally** | Free | VirtualBox / VMware | Simulate multi-node |

---

### Install k3s on a single server

k3s installs as a **single binary** directly on the OS — no Docker involved. It bundles the API server, scheduler, controller, kubelet, and containerd all in one.

```bash
# SSH into your server first, then run:
curl -sfL https://get.k3s.io | sh

# That's it. k3s is now running as a systemd service.
# Check it:
sudo systemctl status k3s

# Use kubectl (k3s includes its own):
sudo k3s kubectl get nodes
```

To use your regular `kubectl` from your laptop, copy the kubeconfig:

```bash
# On the server — print the kubeconfig (replace <SERVER_IP> with your actual IP)
sudo cat /etc/rancher/k3s/k3s.yaml

# Copy that output to your laptop at ~/.kube/config
# Then replace '127.0.0.1' in the file with your server's public IP
# Now from your laptop:
kubectl get nodes   # talks to your real k3s server
```

---

### Add a second node (multi-node cluster)

This is where it gets real — you now have an actual distributed cluster.

```bash
# On the SERVER — get the join token
sudo cat /var/lib/rancher/k3s/server/node-token

# On the AGENT (second machine) — join it to the cluster
curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_IP>:6443 \
  K3S_TOKEN=<TOKEN_FROM_ABOVE> sh -

# Back on your laptop — verify both nodes appear
kubectl get nodes
# NAME        STATUS   ROLES                  AGE
# server-01   Ready    control-plane,master   5m
# agent-01    Ready    <none>                 1m
```

---

### What to practice on your k3s cluster

Now re-do the same labs but on real infrastructure:

| Lab | What changes on real k3s |
|---|---|
| **Lab 06 — Ingress** | Traffic comes from the real internet, not localhost |
| **Lab 08 — HPA** | Real CPU pressure, real autoscaling decisions |
| **Lab 09 — Node failure** | Shut down your agent VM/Pi — watch Kubernetes recover |
| **Lab 10 — ArgoCD** | Point ArgoCD at a real GitHub repo, deploy for real |
| **Lab 12 — Monitoring** | Prometheus scraping real system metrics |

---

### k3d vs k3s — the difference you'll feel

| | k3d (labs) | k3s (real) |
|---|---|---|
| Nodes are | Docker containers on your laptop | Real machines / VMs |
| Network | Simulated | Real network interfaces |
| Storage | Ephemeral | Real disks, real PVs |
| Ingress traffic | `localhost` only | Public internet (with a domain) |
| Node failure | `k3d node stop` command | Physically power off a machine |
| What you learn | Kubernetes concepts & APIs | Operations, networking, real debugging |

> **The Kubernetes API you learned in these labs is 100% identical on k3s.**
> Every YAML file, every `kubectl` command — works exactly the same.
> The only difference is that the consequences are real. 🚀
