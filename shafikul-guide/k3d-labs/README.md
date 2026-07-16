# 🚀 Kubernetes with k3d — Scenario-Based Learning Guide

> **Your machine:** Core i3, 12GB RAM — perfectly fine for all labs here.
> **Approach:** Every lab gives you a **real problem**. You figure it out. Then the solution is below.

---

## Prerequisites (Do This Once)

```bash
# 1. Install Docker
sudo apt update && sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker

# 2. Install k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# 3. Install kubectl
sudo snap install kubectl --classic

# 4. Create your practice cluster
k3d cluster create practice \
  --servers 1 \
  --agents 2 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"

# 5. Verify
kubectl get nodes
# You should see 3 nodes: 1 server + 2 agents
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

```bash
# Cluster management
k3d cluster list
k3d cluster stop practice
k3d cluster start practice
k3d cluster delete practice

# Node management
k3d node list
k3d node stop k3d-practice-agent-0    # simulate node failure
k3d node start k3d-practice-agent-0   # bring it back

# Debugging pods
kubectl get pods -A                    # all namespaces
kubectl describe pod <name>            # full details
kubectl logs <pod-name>                # app logs
kubectl logs <pod-name> --previous     # logs from crashed pod
kubectl exec -it <pod-name> -- bash    # shell into pod
kubectl get events --sort-by=.lastTimestamp  # cluster events

# Quick cleanup
kubectl delete all --all               # delete everything in current namespace
```
