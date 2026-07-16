# 🟢 Lab 02 — Black Friday Traffic Spike

**Level:** Beginner
**Time:** ~20 minutes

---

## 🎬 Scenario

It's 11:58 PM. Black Friday starts in 2 minutes.

Your manager calls you:

> *"Our webapp is running with 2 replicas. Last year we got 50x normal traffic and everything crashed. Scale it up RIGHT NOW. Also set it so K8s automatically handles scaling going forward."*

You have 2 minutes. The app is already running in the `production` namespace.

---

## 📋 Setup (Apply This First)

```bash
kubectl create namespace production

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 2
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
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

Verify it's running:
```bash
kubectl get pods -n production
# 2 pods running
```

**Now solve the problem. Don't read below yet.**

---

## 💡 Hints

<details>
<summary>Hint 1 — Quick manual scaling</summary>

There's a single kubectl command to change replica count without editing YAML.
```bash
kubectl scale --help
```
</details>

<details>
<summary>Hint 2 — Where are pods running?</summary>

You have 3 nodes (1 server + 2 agents). Check which node each pod is on:
```bash
kubectl get pods -n production -o wide
```
</details>

<details>
<summary>Hint 3 — Automatic scaling resource</summary>

Look up `HorizontalPodAutoscaler`. It watches CPU usage and scales automatically.
```bash
kubectl autoscale --help
```
</details>

---

## ✅ Full Solution

### Part 1: Immediate Manual Scale (The Emergency)

```bash
# Scale to 10 replicas RIGHT NOW
kubectl scale deployment webapp -n production --replicas=10

# Watch pods come up in real time
kubectl get pods -n production -w

# Check which nodes pods are distributed across
kubectl get pods -n production -o wide
```

You'll see pods spread across your 2 agent nodes — real load distribution.

---

### Part 2: Automatic Scaling (The Long-Term Fix)

```yaml
# Save as hpa-webapp.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3        # Never go below 3 (always have some headroom)
  maxReplicas: 20       # Never exceed 20 (cost control)
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60   # Scale up when avg CPU > 60%
```

```bash
kubectl apply -f hpa-webapp.yaml

# Check HPA status
kubectl get hpa -n production

# Expected:
# NAME         REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS
# webapp-hpa   Deployment/webapp   5%/60%    3         20        3
```

---

## 🔍 What You Learned

| Concept | Command | When To Use |
|---------|---------|-------------|
| Manual scale | `kubectl scale deployment` | Emergency, immediate need |
| Auto scale | `HorizontalPodAutoscaler` | Production, handles traffic automatically |
| Watch pods | `kubectl get pods -w` | Real-time observation |
| Pod placement | `kubectl get pods -o wide` | See which node each pod is on |

---

## 🔥 Bonus Challenges

**1. Simulate uneven node load:**
```bash
# Stop one agent node — watch pods reschedule
k3d node stop k3d-practice-agent-0
kubectl get pods -n production -o wide -w

# Bring it back
k3d node start k3d-practice-agent-0
```

**2. What happens if you scale to more pods than your nodes can handle?**
```bash
kubectl scale deployment webapp -n production --replicas=50
kubectl get pods -n production
# Some will be "Pending" — why? Check with describe
kubectl describe pod <pending-pod-name> -n production
```

**3. Scale back down after the event:**
```bash
kubectl scale deployment webapp -n production --replicas=3
```

```bash
# Cleanup
kubectl delete namespace production
```
