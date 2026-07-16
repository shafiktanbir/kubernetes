# 🔴 Lab 08 — App Auto-Scales Based on CPU Load

**Level:** Advanced
**Time:** ~40 minutes

---

## 🎬 Scenario

Your e-commerce site is unpredictable. Traffic can go 10x in minutes (flash sales, viral posts).

Your manager says:

> *"I don't want to wake up at 3am to manually scale. The system should scale itself when load increases and scale back down when it's quiet to save costs."*

**Your task:** Set up Horizontal Pod Autoscaler (HPA) so pods scale 2→20 based on CPU.

---

## 📋 Setup

First, install the Metrics Server (required for HPA):

```bash
# k3d comes with metrics-server, but verify it's running
kubectl get deployment metrics-server -n kube-system

# If not running:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch it for k3d (needed for local TLS)
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait for it to be ready
kubectl rollout status deployment metrics-server -n kube-system
```

Deploy the load test app:

```bash
kubectl create namespace production

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"       # HPA bases scaling on THIS number
          limits:
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  namespace: production
spec:
  selector:
    app: php-apache
  ports:
  - port: 80
EOF
```

**Now set up auto-scaling without reading the solution.**

---

## 💡 Hints

<details>
<summary>Hint 1 — Create HPA</summary>

```bash
kubectl autoscale deployment php-apache \
  --cpu-percent=50 \
  --min=2 \
  --max=20 \
  -n production
```
</details>

<details>
<summary>Hint 2 — Generate CPU load</summary>

Run a load generator in a separate pod to spike CPU:
```bash
kubectl run load-gen --image=busybox -n production \
  --restart=Never -it --rm \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```
</details>

<details>
<summary>Hint 3 — Watch HPA in action</summary>

```bash
kubectl get hpa -n production -w
# Watch replicas count increase
```
</details>

---

## ✅ Full Solution

### Step 1: Create the HPA

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2          # Never drop below 2 (always have backup)
  maxReplicas: 20         # Never exceed 20 (cost control)
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50   # Scale when avg CPU > 50% of request
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s before scaling up again
      policies:
      - type: Pods
        value: 4                        # Add max 4 pods at a time
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5min before scaling down
      policies:
      - type: Pods
        value: 2                        # Remove max 2 pods at a time
        periodSeconds: 60
```

```bash
kubectl apply -f hpa.yaml

# Check current state
kubectl get hpa -n production
# NAME             REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
# php-apache-hpa   Deployment/php-apache 0%/50%    2         20        2
```

### Step 2: Generate Load (Terminal 1)

```bash
# Run load generator
kubectl run load-gen \
  --image=busybox \
  -n production \
  --restart=Never \
  -it --rm \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

### Step 3: Watch Auto-Scaling (Terminal 2)

```bash
# Watch HPA metrics update
kubectl get hpa -n production -w

# Watch pods being created
kubectl get pods -n production -w

# Expected sequence:
# REPLICAS: 2 → 4 → 8 → 12 (as CPU climbs above 50%)
```

### Step 4: Stop Load — Watch Scale Down

```bash
# Stop load generator (Ctrl+C in Terminal 1)

# HPA will wait 5 minutes (stabilizationWindowSeconds: 300)
# Then gradually reduce replicas back to 2
kubectl get hpa -n production -w
# REPLICAS: 12 → 10 → 8 → 6 → 4 → 2
```

---

## 🔍 Why the Behavior Settings Matter

```
scaleUp.stabilizationWindowSeconds: 60
→ "Don't panic and add 100 pods in 10 seconds"
→ Wait and see if the spike is real

scaleDown.stabilizationWindowSeconds: 300
→ "Don't scale down too fast after a spike"
→ Traffic might come back — avoid thrashing

policies.value: 4 pods per 60s (scaleUp)
→ Gradual increase, not sudden flood
```

---

## 🔥 Bonus Challenges

**1. Scale based on memory too:**
```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 70
```

**2. What if minReplicas = maxReplicas?**
```yaml
minReplicas: 5
maxReplicas: 5
# HPA is effectively disabled — fixed at 5
```

**3. Disable auto-scaling temporarily (during maintenance):**
```bash
kubectl annotate hpa php-apache-hpa \
  autoscaling.alpha.kubernetes.io/pause=true \
  -n production
```

```bash
# Cleanup
kubectl delete namespace production
```
