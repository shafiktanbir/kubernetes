# 🔴 Lab 09 — A Node Dies. What Happens?

**Level:** Advanced
**Time:** ~40 minutes

---

## 🎬 Scenario

It's 2am. A physical server in your data center loses power. One of your K8s nodes is dead.

Your manager calls:

> *"Node 2 is down! Are our apps still running? How long until everything recovers? Why did some pods NOT recover automatically?"*

**Your task:** Simulate a node failure, observe the behavior, understand why some workloads survive and others don't, and set up **Pod Disruption Budgets** to protect critical apps.

---

## 📋 Setup

```bash
kubectl create namespace production

# Deploy webapp with 4 replicas (spread across nodes)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
EOF

# Deploy a StatefulSet (behaves differently during node failure)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: production
spec:
  serviceName: "db"
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: nginx:1.25
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
EOF
```

Check pod distribution:
```bash
kubectl get pods -n production -o wide
# Note which pods are on which nodes
```

**Now kill a node. Observe. Don't read the solution yet.**

---

## 💡 Hints

<details>
<summary>Hint 1 — Simulate node failure</summary>

```bash
# List your nodes first
k3d node list

# Stop one worker node
k3d node stop k3d-practice-agent-0
```
</details>

<details>
<summary>Hint 2 — Why does recovery take time?</summary>

K8s doesn't immediately declare a node "dead". It waits to avoid false positives.
Check the node status and look at timeouts:
```bash
kubectl get nodes
kubectl describe node k3d-practice-agent-0
```
</details>

<details>
<summary>Hint 3 — StatefulSet vs Deployment during failure</summary>

StatefulSets are more conservative about rescheduling. Research why.
```bash
kubectl get pods -n production -o wide -w
```
</details>

---

## ✅ Full Solution

### Step 1: Verify Pod Distribution Before Failure

```bash
kubectl get pods -n production -o wide
# NAME                      NODE
# webapp-xxx-aaa            k3d-practice-agent-0
# webapp-xxx-bbb            k3d-practice-agent-1
# webapp-xxx-ccc            k3d-practice-agent-0
# webapp-xxx-ddd            k3d-practice-agent-1
# db-0                      k3d-practice-agent-0
# db-1                      k3d-practice-agent-1
# db-2                      k3d-practice-agent-0
```

### Step 2: Kill the Node

```bash
# Open a watch window first (Terminal 2)
kubectl get pods -n production -o wide -w

# Kill node (Terminal 1)
k3d node stop k3d-practice-agent-0
```

### Step 3: Observe Recovery Timeline

```bash
# Immediately after kill:
kubectl get nodes
# NAME                        STATUS
# k3d-practice-server-0       Ready
# k3d-practice-agent-0        Ready      ← Still shows Ready! (takes time)
# k3d-practice-agent-1        Ready

# After ~40 seconds:
# k3d-practice-agent-0        NotReady   ← Node marked NotReady

# After ~5 minutes (default pod-eviction-timeout):
# Deployment pods reschedule automatically to agent-1
# StatefulSet pods stay in "Terminating" — WHY?
```

### Step 4: Why StatefulSets Are Stuck

```bash
kubectl get pods -n production -o wide
# db-0    Terminating   k3d-practice-agent-0   ← STUCK
# db-1    Running       k3d-practice-agent-1
# db-2    Terminating   k3d-practice-agent-0   ← STUCK

# StatefulSets won't reschedule until the node comes back
# OR you manually force delete the pods
```

**Why?** StatefulSets guarantee stable network identity. If db-0 rescheduled to a new node and the old node came back, you'd have **two db-0 pods** — split brain for databases. K8s is being conservative.

```bash
# Force reschedule StatefulSet pods (only if you're sure node is truly dead)
kubectl delete pod db-0 db-2 -n production --force --grace-period=0

# Now they reschedule to agent-1
kubectl get pods -n production -o wide
```

### Step 5: Set Up Pod Disruption Budget (PDB)

Protect your app during voluntary disruptions (node drains, upgrades):

```yaml
# pdb-webapp.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: production
spec:
  minAvailable: 2      # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: webapp
```

```bash
kubectl apply -f pdb-webapp.yaml

# Test PDB during node drain (voluntary disruption)
kubectl drain k3d-practice-agent-1 \
  --ignore-daemonsets \
  --delete-emptydir-data

# K8s will respect PDB: won't drain if it means fewer than 2 pods are running
# You'll see: "Cannot evict pod as it would violate the pod's disruption budget"
```

### Step 6: Bring Node Back

```bash
k3d node start k3d-practice-agent-0

# Watch node recover
kubectl get nodes -w

# After node is Ready:
kubectl get pods -n production -o wide
# Pods do NOT automatically move back to agent-0
# They stay on agent-1 (rebalancing is manual or done by Descheduler)
```

---

## 🔍 Recovery Timeline Summary

```
0s    Node loses power
40s   K8s marks node NotReady (node-monitor-grace-period)
5min  K8s starts evicting pods from dead node
5min  Deployment pods → reschedule immediately to healthy nodes
5min  StatefulSet pods → stuck in Terminating (by design)

Manual action needed for StatefulSets:
      kubectl delete pod --force --grace-period=0
```

---

## 🔍 What You Learned

| Scenario | Deployment | StatefulSet |
|---------|-----------|-------------|
| Node dies | Auto-reschedule ✅ | Stuck until force-deleted ⚠️ |
| Node comes back | Pods stay where they are | Pods stay where they are |
| Voluntary drain | Respects PDB ✅ | Respects PDB ✅ |
| Best for | Stateless apps | Databases, message queues |

---

## 🔥 Bonus Challenges

**1. Use node affinity to spread pods across nodes:**
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: ["webapp"]
      topologyKey: kubernetes.io/hostname
# Forces pods to be on DIFFERENT nodes
```

**2. What's the default eviction timeout?**
```bash
kubectl get nodes -o yaml | grep taint
# Look for node.kubernetes.io/not-ready:NoExecute with tolerationSeconds
```

**3. Cordon a node (mark unschedulable without evicting):**
```bash
kubectl cordon k3d-practice-agent-0
# New pods won't schedule here, existing pods stay
kubectl uncordon k3d-practice-agent-0
```

```bash
# Cleanup
kubectl delete namespace production
kubectl delete pdb webapp-pdb -n production 2>/dev/null
```
