# ⭐⭐ Troubleshooting — The Most Important Interview Topic

> These questions separate people who read about K8s from people who've actually used it.
> Interviewers love these because they reveal real experience.

---

## Q1: A pod is in CrashLoopBackOff. Walk me through exactly how you debug it.

**Senior Answer:**

CrashLoopBackOff means the container is starting, crashing, and K8s is restarting it with exponential backoff. I'd systematically narrow down the cause:

**Step 1 — Get the exit code:**
```bash
kubectl describe pod <pod-name> -n <namespace>
# Look for "Last State: Terminated"
# Exit Code: 1  → app crashed (non-zero exit)
# Exit Code: 137 → OOMKilled (memory limit exceeded)
# Exit Code: 139 → Segmentation fault
# Exit Code: 143 → SIGTERM not handled (graceful shutdown issue)
```

**Step 2 — Read the logs from the crashed container:**
```bash
kubectl logs <pod-name> -n <namespace>            # current container
kubectl logs <pod-name> -n <namespace> --previous # logs from crashed instance
```

**Step 3 — Common root causes by exit code:**

- **Exit 1, OOMKilled** → increase memory limits, check for memory leaks
- **Exit 1, app startup logs say "connection refused"** → database/dependency not ready, app crashes on startup. Fix: add readinessProbe, or use initContainers to wait for DB.
- **Exit 1, "permission denied"** → container running as non-root can't access mounted volumes or files
- **"Back-off restarting failed container"** with no logs → image starts but exits immediately. Common when CMD is wrong in Dockerfile.

**Step 4 — If logs are empty:**
```bash
# Try to exec into a debug container
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
# Or for a completed/crashed pod, check events
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

**What I'd say to stand out:** "I've found that 80% of CrashLoopBackOff issues are either bad startup config (can't connect to DB), OOMKill, or a misconfigured entrypoint. The other 20% are app bugs you need logs to diagnose."

---

## Q2: A pod is stuck in Pending state. Why could that be?

**Senior Answer:**

Pending means the scheduler can't find a suitable node. I'd check in this order:

**Check 1 — Scheduler reason:**
```bash
kubectl describe pod <pod-name> -n <namespace>
# Events section at the bottom is key
# Common messages:
# "0/3 nodes are available: 3 Insufficient cpu"
# "0/3 nodes are available: 3 Insufficient memory"
# "0/3 nodes are available: 3 node(s) had untolerated taint"
# "0/3 nodes are available: 1 node(s) didn't match node selector"
```

**Root causes:**

1. **Insufficient resources** — the pod's `requests` exceed what any node has available. Fix: scale the cluster, reduce requests, or check if other pods are wasting resources.

2. **Node selector / affinity not matching** — pod has `nodeSelector: zone: us-east-1a` but no nodes have that label.

3. **Taint not tolerated** — nodes have a taint that the pod doesn't tolerate. Common with GPU nodes — they're tainted `nvidia.com/gpu=present:NoSchedule` so only GPU pods that explicitly tolerate it can schedule there.

4. **PVC not bound** — pod needs a PersistentVolumeClaim but the PVC is stuck Pending. Check storage class, check if the PV was provisioned.

5. **Too many pods per node** — default limit is 110 pods per node. Rarely hit but it happens.

**The answer that impresses:** "The first thing I do is `kubectl describe pod` and read the Events section. The scheduler always leaves a reason. Then I check `kubectl get nodes` to see if nodes are Ready, and `kubectl describe node` to see available allocatable resources."

---

## Q3: Your deployment updated successfully but users are still seeing the old version. Why?

**Senior Answer:**

Classic scenario. A few possible causes:

**1. Caching — most common**
- Browser cache serving old JS/CSS
- CDN (CloudFront, Cloudflare) caching old responses
- Fix: cache-busting headers, invalidate CDN cache, versioned asset filenames

**2. Old pods still serving traffic**
The rolling update replaced pods, but the Service is still load-balancing to the old ones during the transition. After the rollout completes:
```bash
kubectl rollout status deployment/webapp
kubectl get pods   # verify all new pods are Running
```

**3. Readiness probe passing before app is ready**
If the readinessProbe path returns 200 too quickly (before the app finishes initialization), K8s adds the pod to the Service endpoints before the new code is fully loaded. Fix: make the readiness probe more rigorous — check if the app has completed startup (loaded config, warmed up DB connections, etc.)

**4. Wrong namespace**
You deployed to `staging`, not `production`. I've seen this happen.
```bash
kubectl get deployment webapp -n production -o yaml | grep image
```

**5. Image tag not updated**
You used `latest` tag. K8s doesn't re-pull an image if it's already cached on the node and `imagePullPolicy` is not `Always`. Fix: use immutable tags like `v2.3.1` or SHA digests instead of `latest`.

---

## Q4: How do you debug a service that's not routing traffic to pods?

**Senior Answer:**

Networking issues in K8s have a clear debug path:

**Step 1 — Verify the Service exists and has endpoints:**
```bash
kubectl get service webapp-service -n production
kubectl get endpoints webapp-service -n production
# If ENDPOINTS shows <none> — pods aren't matching the service selector
```

**Step 2 — Check if selectors match:**
```bash
# Get Service selector
kubectl get svc webapp-service -n production -o yaml | grep selector -A5

# Get Pod labels
kubectl get pods -n production --show-labels

# The selector MUST exactly match pod labels — typos break this silently
```

**Step 3 — Test connectivity from inside the cluster:**
```bash
# Run a debug pod
kubectl run debug --image=busybox -it --rm -- sh

# Inside the pod:
wget -O- http://webapp-service.production.svc.cluster.local
# Tests: DNS resolution + Service routing + Pod connectivity
```

**Step 4 — Test direct Pod IP:**
```bash
kubectl get pod -n production -o wide   # get pod IP
# From debug pod:
wget -O- http://<pod-ip>:80
# If this works but Service doesn't → issue is in Service/kube-proxy
# If this doesn't work → issue is in the pod/container itself
```

**Step 5 — Check kube-proxy:**
```bash
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs <kube-proxy-pod> -n kube-system | grep error
```

**The answer that stands out:** "I always separate: is this a DNS issue, a Service selector issue, or a Pod issue? Testing the pod IP directly bypasses the Service and tells me immediately which layer is broken."

---

## Q5: What's the difference between Liveness and Readiness probes? What happens if you get them wrong?

**Senior Answer:**

**Readiness probe** — tells K8s "is this pod ready to receive traffic?" If it fails, the Pod is removed from the Service's endpoints. Traffic stops going to it. The pod is NOT restarted — it just goes "out of rotation" until the probe passes again.

**Liveness probe** — tells K8s "is this pod still alive?" If it fails, K8s kills and restarts the container. This is for detecting deadlocks or stuck states where the process is running but not functional.

**Startup probe** — tells K8s "has the app finished starting up?" Used for slow-starting apps to prevent the liveness probe from killing a container that's still loading.

**What happens when you get them wrong:**

*Bad liveness probe (too aggressive):*
```yaml
livenessProbe:
  httpGet:
    path: /health
  initialDelaySeconds: 5   # Too short for a slow-starting app
  failureThreshold: 1       # Too low — one slow response = restart
```
Result: K8s restarts your pod every time it has a brief slowdown. Database query takes 2 seconds during a spike → liveness fails → pod restarts → pod takes 30s to restart → users get errors → K8s kills it again → CrashLoopBackOff. This is a common self-inflicted outage.

*Missing readiness probe:*
During a rolling update, new pods start receiving traffic immediately when containers start — before your app has finished initializing (loading config, DB migrations, warming caches). Users hit the new pods and get 500 errors for 10-30 seconds. Readiness probes fix this by keeping pods out of rotation until they're actually ready.

**Rule of thumb:** Readiness probe = "are you ready to serve users?" Liveness probe = "are you alive at all?" Set generous `initialDelaySeconds` and `failureThreshold` for liveness probes.

---

## Q6: A node shows NotReady. What do you check?

**Senior Answer:**

```bash
# Step 1: Check node condition
kubectl describe node <node-name>
# Look at "Conditions" section:
# Ready: False — kubelet is not reporting healthy
# MemoryPressure: True — node is running out of memory
# DiskPressure: True — disk nearly full
# PIDPressure: True — too many processes

# Step 2: Check kubelet on the node (if you have SSH access)
systemctl status kubelet
journalctl -u kubelet -n 100  # last 100 lines of kubelet logs

# Step 3: Check if it's a networking issue
# Can the control plane reach the node?
# Is the container runtime (containerd) running?
systemctl status containerd

# Step 4: Check node resources
kubectl top node <node-name>
df -h   # disk usage on the node
free -h # memory on the node
```

**Common causes:**
- Kubelet crashed (restart it: `systemctl restart kubelet`)
- containerd crashed (restart: `systemctl restart containerd`)
- Node ran out of disk space (eviction kicks in, node becomes NotReady)
- Network partition between node and control plane
- Node was terminated by autoscaler or cloud provider

**What K8s does automatically:** After `node-monitor-grace-period` (default 40s), the node is marked NotReady. After `pod-eviction-timeout` (default 5min), pods on that node are evicted and rescheduled.

---

## Q7: How do you debug high latency in a service running on Kubernetes?

**Senior Answer:**

I approach this systematically, ruling out layers:

**1. Is it all pods or one pod?**
```bash
kubectl top pods -n production
# Is one pod using 95% CPU while others are at 10%?
# That pod is the bottleneck — kill it, let it reschedule
```

**2. Is it resource throttling?**
```bash
# Check if containers are being CPU throttled
kubectl describe pod <name> -n production | grep -A10 "Limits"
# If requests are set too low → CPU throttling → latency
```

**3. Is it a downstream dependency?**
```bash
# Exec into the pod and test dependencies
kubectl exec -it <pod-name> -- sh
curl -w "%{time_total}" http://database-service:5432  # measure DB response time
curl -w "%{time_total}" http://redis-service:6379
```

**4. Is it pod-to-pod network latency?**
In multi-node clusters, pod communication across nodes goes through the CNI overlay network. This adds latency vs. same-node communication. Pod anti-affinity spreading pods across nodes can increase latency for chatty services.

**5. Look at metrics:**
```bash
# If Prometheus is installed
kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090
# Query: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
# This shows p99 latency per endpoint
```

**What I'd say to impress:** "High latency in K8s is rarely a K8s problem itself — it's usually resource limits causing CPU throttling, an overloaded downstream dependency, or GC pauses in the app. K8s is just surfacing a problem that exists in the application or infrastructure."
