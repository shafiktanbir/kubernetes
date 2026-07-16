# 🔴 Lab 12 — App is Slow. Find Out Why.

**Level:** Advanced
**Time:** ~50 minutes

---

## 🎬 Scenario

Users are complaining. Response times are slow. Your on-call phone rings.

> *"The webapp is taking 8 seconds to respond. We have no idea why. CPU? Memory? Too many requests? A pod crashing and restarting? We're blind."*

Your task: **Install Prometheus + Grafana, instrument your cluster, and find the bottleneck using real dashboards.**

---

## 📋 Prerequisites

```bash
# Helm must be installed (from Lab 11)
helm version
```

---

## ✅ Full Solution

### Step 1: Install Prometheus + Grafana Stack

```bash
# Add the Prometheus community Helm repo
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

# Install the full monitoring stack
# (Prometheus + Grafana + AlertManager + Node Exporter + kube-state-metrics)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=24h

# Wait for all pods (takes 3-5 minutes)
kubectl get pods -n monitoring -w
# When all pods show Running, press Ctrl+C
```

### Step 2: Access Grafana Dashboard

```bash
# Port-forward Grafana
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80 &

# Open browser: http://localhost:3000
# Username: admin
# Password: admin123
```

In Grafana, click **Dashboards → Browse → Kubernetes**. You'll see pre-built dashboards for:
- Cluster Overview
- Node metrics (CPU, RAM, Disk)
- Pod metrics
- Deployment status

### Step 3: Deploy a Misbehaving App

```bash
kubectl create namespace production

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
      annotations:
        prometheus.io/scrape: "true"    # Tell Prometheus to scrape this pod
        prometheus.io/port: "80"
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"    # ← We'll make this too low on purpose
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  namespace: production
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
EOF
```

### Step 4: Generate Load to Trigger Issues

```bash
# Install a load generator
kubectl run load-gen \
  --image=busybox \
  -n production \
  --restart=Never \
  -it --rm \
  -- /bin/sh -c "while true; do \
      wget -q -O- http://webapp-service && \
      wget -q -O- http://webapp-service && \
      wget -q -O- http://webapp-service; \
    done"
```

### Step 5: Monitor Using kubectl (Without Grafana)

```bash
# CPU and Memory usage of pods
kubectl top pods -n production
# NAME               CPU(cores)   MEMORY(bytes)
# webapp-xxx-aaa     45m          20Mi
# webapp-xxx-bbb     78m          20Mi
# webapp-xxx-ccc     120m         22Mi   ← this one is hot!

# CPU usage of nodes
kubectl top nodes
# NAME                    CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# k3d-practice-agent-0   234m         11%    512Mi           42%
# k3d-practice-agent-1   890m         44%    1024Mi          85%   ← stressed!

# Watch in real time
watch kubectl top pods -n production
```

### Step 6: Identify OOMKilled Pod

```bash
# Simulate a memory issue — set limits too low
kubectl set resources deployment webapp -n production \
  --limits=memory=5Mi   # Absurdly low — will OOMKill

# Watch pods crash and restart
kubectl get pods -n production -w
# webapp-xxx-aaa    0/1   OOMKilled   3   30s   ← it crashes!

# Read the crash reason
kubectl describe pod <crashed-pod-name> -n production | grep -A5 "Last State"
# Last State:   Terminated
#   Reason:     OOMKilled       ← memory limit too low
#   Exit Code:  137

# Check restart count
kubectl get pods -n production
# NAME             READY   STATUS             RESTARTS
# webapp-xxx-aaa   0/1     CrashLoopBackOff   5         ← restarting over and over
```

```bash
# Fix: increase memory limit
kubectl set resources deployment webapp -n production \
  --limits=memory=128Mi \
  --requests=memory=64Mi
```

### Step 7: Use Grafana to Find the Problem

In your browser at http://localhost:3000:

1. **Kubernetes / Compute Resources / Pod** dashboard
   - Find pods with high CPU % (> 80% of limit = danger zone)
   - Find pods with high memory % (> 90% = about to OOMKill)

2. **Kubernetes / Compute Resources / Node** dashboard
   - Which node is under pressure?
   - Is one node doing all the work?

3. **Kubernetes / Workloads / Deployment** dashboard
   - Restart count graph → spikes = CrashLoopBackOff events
   - Availability graph → dips = pods were unavailable

---

## 🔍 Key Metrics to Monitor in Production

| Metric | Alert Threshold | What It Means |
|--------|----------------|---------------|
| Pod CPU > 80% of limit | Warning | App approaching limit, scale up |
| Pod memory > 90% of limit | Critical | OOMKill imminent |
| Pod restarts > 5 in 1hr | Critical | CrashLoopBackOff |
| Node CPU > 70% | Warning | Node overloaded |
| Pod pending > 5min | Warning | Not enough resources to schedule |

### Step 8: Set Up an Alert Rule

```yaml
# alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: webapp-alerts
  namespace: monitoring
  labels:
    release: monitoring    # Must match Prometheus operator labels
spec:
  groups:
  - name: webapp.rules
    rules:
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{namespace="production"}[5m]) * 60 > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Pod has restarted {{ $value }} times in the last 5 minutes"

    - alert: PodMemoryHigh
      expr: |
        container_memory_working_set_bytes{namespace="production"}
        /
        container_spec_memory_limit_bytes{namespace="production"} > 0.9
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} memory above 90%"
```

```bash
kubectl apply -f alert-rules.yaml

# Verify alert rule was picked up
kubectl get prometheusrule -n monitoring
```

---

## 🔍 Monitoring Stack Architecture

```
Your Pods           node-exporter        kube-state-metrics
(app metrics)  →    (node metrics)  →    (K8s object metrics)
      │                   │                      │
      └───────────────────┴──────────────────────┘
                          │
                    Prometheus
                  (collects + stores)
                          │
                       Grafana
                    (visualizes)
                          │
                   AlertManager
                 (sends Slack/email)
```

---

## 🔥 Bonus Challenges

**1. Access Prometheus directly:**
```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 &
# http://localhost:9090
# Try query: rate(http_requests_total[5m])
```

**2. Send alerts to Slack (real production setup):**
```bash
# Edit AlertManager config
kubectl edit secret monitoring-kube-prometheus-alertmanager -n monitoring
# Add your Slack webhook URL
```

**3. Install Loki for logs alongside Prometheus for metrics:**
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  -n monitoring \
  --set grafana.enabled=false \
  --set prometheus.enabled=false
```

```bash
# Cleanup
kubectl delete namespace production
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```
