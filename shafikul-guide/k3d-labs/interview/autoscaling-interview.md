# 🎯 Autoscaling — Senior Engineer Interview Problem Set

> **Format:** Every problem is a real scenario. No "what is HPA?" questions.
> These are the questions that separate engineers who've *read* about autoscaling
> from engineers who've *been paged at 3am because of it*.
>
> **How to use this:**
> - Try to answer each question yourself first
> - The answer is hidden below — reveal only after you've thought it through
> - If you can't answer it, note it down and go build the scenario in your k3d lab

---

## 🔴 Section 1 — The Trap Questions (Conceptual but Sneaky)

These look simple. They're not.

---

### Q1. HPA is configured. Traffic spikes. Pods scale from 3 → 12. Users still see errors for 90 seconds. Why? What would you change?

<details>
<summary>💡 Senior Answer</summary>

**Root cause: HPA is reactive. It cannot prevent the scale-up window.**

The sequence:
```
Spike hits → metrics server polls (15s default) → HPA evaluates (30s cycle)
→ scheduler places pods → image pull (if not cached) → container starts
→ app initializes → readiness probe passes → pod receives traffic

Total: 60–120 seconds minimum, even with a healthy cluster
```

**Fixes (in order of impact):**

1. **Raise `minReplicas`** — the real fix. If you know your baseline is 5 pods, don't run 3. The cheapest replicas are the ones already running when the spike hits.

2. **Optimize startup time** — if your app takes 45s to start (JVM warmup, DB migrations, etc.), fix that first. Target < 10s.

3. **Tune readiness probe** — don't use `initialDelaySeconds: 30` as a lazy workaround for slow startups. Use proper health endpoints.

4. **Pre-scale on schedule** — if traffic is predictable (morning rush, end-of-month billing runs), use a CronJob or KEDA scheduled trigger to scale up *before* the spike, not after.

5. **Adjust HPA sync period** — `--horizontal-pod-autoscaler-sync-period=10s` (default 15s) on the controller-manager.

**What NOT to do:** lower `averageUtilization` threshold to 30% so it scales earlier. That just means you're always running 3x the pods you need.

</details>

---

### Q2. Your HPA has `minReplicas: 1`. What production problem does this create that has nothing to do with traffic?

<details>
<summary>💡 Senior Answer</summary>

**Zero-downtime deployments become impossible.**

When you do a rolling update with 1 replica:
- Kubernetes terminates the old pod
- Starts the new pod
- During the transition: **0 pods serving traffic**

Even with `maxUnavailable: 0`, if you only have 1 replica, the math doesn't work — Kubernetes can't maintain 1 running while replacing 1.

**Other problems with `minReplicas: 1`:**
- Node maintenance/drain: your single pod gets evicted → downtime
- Pod crash: app is completely down until Kubernetes restarts it (could be 30+ seconds)
- No redundancy: single point of failure defeats the entire point of running in Kubernetes

**Rule:** `minReplicas` should never be 1 for any production workload. Minimum is 2, ideally spread across zones with a `topologySpreadConstraint`.

</details>

---

### Q3. You set `averageUtilization: 70` for CPU. Your app is running at exactly 70% CPU on all 5 pods. Will HPA scale up or down? Will it do anything at all?

<details>
<summary>💡 Senior Answer</summary>

**It will do nothing — and that's correct behavior.**

HPA uses this formula:
```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))
desiredReplicas = ceil(5 × (70 / 70)) = ceil(5 × 1.0) = ceil(5.0) = 5
```

No change needed. HPA has a built-in **tolerance band of ±10%** (configurable via `--horizontal-pod-autoscaler-tolerance`, default `0.1`). If the ratio is between 0.9 and 1.1, HPA won't act — it avoids constant "flapping" (rapid scale up/down).

**Follow-up:** What if CPU is at 76%?
```
desiredReplicas = ceil(5 × (76 / 70)) = ceil(5.43) = 6
```
Now HPA will scale up to 6 pods, because 76/70 = 1.086, which exceeds the 10% tolerance.

</details>

---

### Q4. You have HPA configured. You also manually run `kubectl scale deployment webapp --replicas=20`. What happens next?

<details>
<summary>💡 Senior Answer</summary>

**HPA will fight you and win — eventually.**

HPA continuously reconciles the replica count to match its desired state based on metrics. Within the next sync cycle (15–30 seconds), HPA will:

- Calculate the desired replica count from current CPU/memory
- Override your manual `kubectl scale` with its own value
- Your 20 replicas will be reduced (if CPU doesn't justify 20)

**This means:** manual `kubectl scale` is **useless** and potentially misleading when HPA is active.

**What you should do instead:**
- Temporarily patch the HPA: `kubectl patch hpa webapp-hpa -p '{"spec":{"minReplicas":20}}'`
- Or delete the HPA, scale manually, then recreate it
- Or use a proper deployment freeze/scale-lock mechanism

**Lesson:** never manually scale a deployment that has an HPA. You'll confuse yourself and your teammates.

</details>

---

### Q5. HPA is configured with CPU metrics. CPU is at 20%. Users are reporting the app is slow. Your manager says "scale it up." What do you actually do?

<details>
<summary>💡 Senior Answer</summary>

**You don't scale it up — you diagnose first.**

CPU at 20% with slowness means CPU is NOT the bottleneck. Scaling app pods would change nothing and waste money. The real bottleneck is elsewhere:

**Diagnostic steps:**
```bash
# 1. Check if it's a DB issue
kubectl exec -it <pod> -- curl localhost:8080/metrics | grep db_query_duration

# 2. Check pod memory — maybe it's GC pressure (high memory → frequent GC pauses)
kubectl top pods -n production

# 3. Check network — external API call timing out?
kubectl logs <pod> | grep -i "timeout\|slow\|error"

# 4. Check events — OOMKill? Throttling?
kubectl get events -n production --sort-by='.metadata.creationTimestamp'

# 5. Check if it's a single pod or all pods
kubectl get pods -n production  # any restarts? any not-ready?
```

**Common real causes:**
| Symptom | Root Cause |
|---|---|
| Low CPU, slow response | DB slow query, missing index |
| Low CPU, high memory | Memory leak → GC pauses → slow response |
| Low CPU, intermittent slow | External API timeout, no circuit breaker |
| Low CPU, some pods slow | One pod has bad state — restart it |

**Scaling up pods when DB is the bottleneck actively makes it worse** — more pods = more DB connections = more DB load.

</details>

---

## 🔴 Section 2 — YAML Debugging (Find the Bug)

---

### Q6. This HPA has been applied. It never scales. Find all the bugs.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: staging
spec:
  scaleTargetRef:
    kind: Deployment
    name: webapp
  minReplicas: 5
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

<details>
<summary>💡 Senior Answer</summary>

**Bug 1 — `minReplicas == maxReplicas`**
```yaml
minReplicas: 5
maxReplicas: 5  # ← HPA cannot scale if min and max are equal
```
HPA can only scale within the range `[minReplicas, maxReplicas]`. If they're equal, it's effectively a fixed replica count. HPA will still run but will never actually change anything.

**Bug 2 — Missing `scaleTargetRef.apiVersion`**
```yaml
scaleTargetRef:
  kind: Deployment
  name: webapp
  # apiVersion: apps/v1 ← missing
```
Without this, some versions of Kubernetes default to `apps/v1` but it's not guaranteed. Always be explicit.

**Bug 3 — Namespace mismatch (potential)**
The HPA is in `staging` namespace but the Deployment may be in a different namespace. HPA can only target resources in the same namespace as itself.

**Bug 4 — No resource requests on pods (not visible here, but almost always the real issue)**
HPA CPU utilization is calculated as:
```
actualCPU / requestedCPU × 100
```
If your pods have no `resources.requests.cpu` defined, the metrics server returns 0 or errors, and HPA cannot compute utilization. This is the #1 reason HPA appears to "do nothing" in the real world.

Check with:
```bash
kubectl describe hpa webapp-hpa -n staging
# Look for: "FailedGetResourceMetric" or "missing request for cpu"
```

</details>

---

### Q7. This HPA scales up correctly but never scales down even after traffic drops for 30 minutes. Why?

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 3600
```

<details>
<summary>💡 Senior Answer</summary>

**The `stabilizationWindowSeconds: 3600` means it won't scale down for 1 full hour after the last scale event.**

This is a deliberate `behavior` configuration. The stabilization window prevents "thrashing" — rapidly scaling down and then having to scale back up again if traffic is variable. During the window, HPA tracks the *maximum* desired replica count it has seen and uses that as a floor.

A 3600-second (1 hour) window is very aggressive. Common values:
- Scale **up**: `stabilizationWindowSeconds: 0` — scale up immediately, no delay
- Scale **down**: `stabilizationWindowSeconds: 300` — wait 5 minutes before scaling down

**When a 1-hour scale-down window makes sense:** financial trading systems, live event streaming — where you want to aggressively hold capacity after a peak because a re-spike is costly.

**When it doesn't make sense:** most web apps. You're paying for 50 pods for an hour when you need 5.

**Fix for a normal web app:**
```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0        # scale up immediately
    policies:
      - type: Pods
        value: 4                          # add max 4 pods per 60s
        periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300      # wait 5 minutes before scaling down
    policies:
      - type: Percent
        value: 50                         # remove max 50% of pods per 60s
        periodSeconds: 60
```

</details>

---

## 🔴 Section 3 — Production Incident Scenarios

---

### Q8. 3am. PagerDuty fires. "API error rate 40%." You SSH into the bastion. Walk me through your exact diagnostic steps.

<details>
<summary>💡 Senior Answer</summary>

**First 60 seconds — get situational awareness, touch nothing:**

```bash
# 1. How many pods are actually running vs desired?
kubectl get deployment api-server -n production

# 2. Are pods healthy?
kubectl get pods -n production -l app=api-server

# 3. Any recent events — OOMKill? CrashLoop? Image pull fail?
kubectl get events -n production --sort-by='.metadata.creationTimestamp' | tail -20

# 4. What did HPA decide recently?
kubectl describe hpa api-server-hpa -n production

# 5. Are pods actually receiving traffic or is it a service/ingress issue?
kubectl get endpoints api-service -n production

# 6. Is this ALL pods failing or just some?
kubectl top pods -n production -l app=api-server
```

**Next 2 minutes — look at logs of a failing pod:**
```bash
kubectl logs <failing-pod> -n production --tail=100
kubectl logs <failing-pod> -n production --previous  # if it crashed
```

**Decision tree:**
```
Error rate 40%
├── 0 pods running → PodDisruptionBudget blocking drain? Node issue? Image pull fail?
├── Pods running but crashing → CrashLoopBackOff → check logs --previous
├── Pods running and healthy → Service/Ingress misconfiguration → check endpoints
├── Pods running, high memory → OOM → add memory limit, check for leak
└── Pods running, DB errors in logs → DB is the problem, not the app
```

**Golden rule:** describe the incident in Slack/incident channel *before* you start making changes. Even at 3am. You want a paper trail, and someone else might have context.

</details>

---

### Q9. You have a web app with HPA. During Black Friday, the system scaled correctly to 80 pods. But then ALL 80 pods went into CrashLoopBackOff simultaneously. What happened and how do you recover?

<details>
<summary>💡 Senior Answer</summary>

**Most likely cause: cascading resource exhaustion.**

When 80 pods start simultaneously, they all try to:
- Open DB connections → DB connection limit exceeded → all pods error → restart → repeat
- Pull the same image → registry rate limit hit
- Read config from the same ConfigMap/Secret/external service → that service gets overwhelmed

**This is a "thundering herd" problem.** Autoscaling caused it by scaling too fast.

**Immediate recovery:**
```bash
# 1. Stop the bleeding — freeze HPA by setting maxReplicas = currentReplicas
kubectl patch hpa webapp-hpa -p '{"spec":{"maxReplicas":80}}'

# 2. Check what the pods are actually crashing on
kubectl logs <any-crashing-pod> -n production --previous

# 3. If it's DB connections — reduce pods immediately to what DB can handle
# Calculate: DB maxConnections / connectionsPerPod = safeReplicas
kubectl scale deployment webapp --replicas=20 -n production
# (HPA will fight this — also patch minReplicas)
kubectl patch hpa webapp-hpa -p '{"spec":{"minReplicas":20,"maxReplicas":20}}'

# 4. Once stable, slowly increase
kubectl patch hpa webapp-hpa -p '{"spec":{"maxReplicas":30}}'
# Wait 5 minutes, verify stable, then increase again
```

**How to prevent this in future:**
1. **Connection pooler** (PgBouncer) — 500 app connections → 20 DB connections
2. **HPA scale-up policy** — limit how fast pods can be added:
   ```yaml
   behavior:
     scaleUp:
       policies:
         - type: Pods
           value: 5        # add max 5 pods per minute, not 50
           periodSeconds: 60
   ```
3. **PodDisruptionBudget** — ensures at least X healthy pods always exist during changes
4. **Startup probe** — prevent pod from receiving traffic until it's truly ready

</details>

---

### Q10. Your company runs a SaaS app. A customer's batch job causes a CPU spike at 2am every night. HPA scales to 40 pods at 2am and back to 5 pods by 4am. What's wrong with this pattern and how do you fix it?

<details>
<summary>💡 Senior Answer</summary>

**The problem: one customer's workload is affecting all customers' costs and potentially stability.**

**Secondary problem:** the 2am HPA reaction is always 2–3 minutes late. During those 3 minutes, other customers see degraded service because the batch job is consuming all the CPU on the existing 5 pods.

**Fix 1: Namespace isolation / resource quotas**
```yaml
# Give the batch customer their own namespace with limits
apiVersion: v1
kind: ResourceQuota
metadata:
  name: batch-quota
  namespace: customer-batch
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    pods: "10"
```

**Fix 2: Pre-scale on a schedule (KEDA or CronJob)**
```yaml
# KEDA scheduled trigger — scale up at 1:55am, down at 4:05am
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  triggers:
    - type: cron
      metadata:
        timezone: "UTC"
        start: "55 1 * * *"   # 1:55am — scale up BEFORE the job
        end:   "5 4 * * *"    # 4:05am — scale down AFTER the job
        desiredReplicas: "40"
```

**Fix 3: Priority Classes — batch job gets lower priority**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low-priority
value: 100          # normal pods: 1000 — batch pods get evicted first
preemptionPolicy: Never
```

**Fix 4: Horizontal vs Vertical split — run batch jobs on separate nodes**
Use node selectors or taints to isolate batch workloads from user-facing services entirely. Different node pool for batch, different node pool for the API.

</details>

---

## 🔴 Section 4 — Architecture Design Questions

---

### Q11. Design the autoscaling strategy for a video transcoding service. It processes jobs from a queue. CPU usage is 5% when idle, 95% when processing. HPA based on CPU — good idea or not?

<details>
<summary>💡 Senior Answer</summary>

**No — CPU-based HPA is the wrong tool here. Use KEDA with queue-length metrics.**

**Why CPU-based HPA fails for queue workers:**
- When queue is empty: CPU 5%, 0 jobs processing → HPA scales DOWN to minReplicas ✓
- When queue has 1000 jobs: CPU spikes to 95% → HPA starts scaling → but it took 60–120 seconds to react → jobs piled up during that window
- CPU is a *lagging indicator* of work to be done. Queue depth is a *leading indicator*.

**The right architecture: KEDA with queue-based scaling**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: transcoder-scaler
spec:
  scaleTargetRef:
    name: video-transcoder
  minReplicaCount: 0    # can scale to zero when queue is empty (save $$$)
  maxReplicaCount: 50
  triggers:
    - type: rabbitmq   # or SQS, Kafka, Redis, etc.
      metadata:
        queueName: transcoding-jobs
        queueLength: "5"  # 1 pod per 5 jobs in queue
```

**Benefits:**
- Scale to **zero** when queue is empty — no idle costs
- Scale up *proportionally to actual work*, not CPU lag
- Pre-emptive: if 100 jobs arrive at once, scale to 20 pods immediately — don't wait for CPU to spike

**Additional considerations:**
- Each transcoder pod should process 1 job at a time (simplest, most predictable)
- Jobs should be idempotent — if a pod is killed mid-job, the job goes back to queue
- Set pod `terminationGracePeriodSeconds` long enough to finish in-progress jobs

</details>

---

### Q12. You're asked to design autoscaling for a stateful service — a Redis cluster. How do you approach this? Can you use HPA?

<details>
<summary>💡 Senior Answer</summary>

**You cannot use HPA directly on a Redis cluster in any meaningful way. This is fundamentally a different problem.**

**Why HPA doesn't work for stateful services:**
- Redis nodes are not interchangeable — each has its own slot range in a cluster
- Adding a new Redis pod doesn't automatically rebalance data to it
- Removing a pod means the data on it is gone (or needs migration first)
- "Scaling" Redis requires cluster-aware resharding, not just spinning up pods

**The right approach: Kubernetes Operators**

Operators encode the operational knowledge (how to scale Redis safely) into a controller:
```bash
# Redis Operator (e.g., Redis Enterprise or Spotahome redis-operator)
kubectl apply -f redis-cluster.yaml
# The operator handles: slot rebalancing, replica promotion, safe scaling
```

**For pure caching (no persistence) — you CAN use HPA:**
If Redis is used only as a cache (data can be lost without consequence), you can treat it as stateless:
- Use `Deployment` not `StatefulSet`
- Use consistent hashing client-side so pods can be added/removed
- HPA on memory utilization is reasonable here

**For primary data store — use vertical scaling first:**
```yaml
# VPA — let Kubernetes right-size the Redis pod automatically
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
spec:
  targetRef:
    name: redis
  updatePolicy:
    updateMode: "Auto"   # WARNING: this restarts pods. Use "Off" to just get recommendations
```

**Senior lesson:** "Can we autoscale this?" is usually the wrong question for stateful services. The right questions are: "What's our capacity ceiling?", "When do we need to plan the next scaling event?", and "What's our data migration strategy?"

</details>

---

## 🔴 Section 5 — The "What Would You Actually Do?" Round

No right or wrong answer — these test your engineering judgment.

---

### Q13. Your CTO says: "Set `maxReplicas: 1000` so we never have scaling issues again." How do you respond?

<details>
<summary>💡 Senior Answer</summary>

**You push back, professionally, with data.**

"I understand the intent — we want to ensure we never hit a capacity ceiling. But `maxReplicas: 1000` creates a different class of risks:

1. **Runaway cost incident:** a load test, a DDoS, or a bug that causes request loops could scale to 1000 pods overnight. At $0.05/pod/hour on our current instance type, that's $50/hour or $1,200/day — for a cluster that can't actually serve that much traffic anyway because our DB maxes out at 100 pods worth of connections.

2. **It doesn't solve the real problem:** if we've been hitting our current maxReplicas, we need to understand *why*. Is it legitimate traffic growth? A bug causing request amplification? A missing cache? Scaling ceiling is a symptom, not the disease.

3. **Cluster capacity:** 1000 pods requires 1000× the CPU/memory requests. Our cluster's nodes don't have that capacity. HPA would scale up, pods would go Pending, and we'd have a different kind of incident.

**What I'd propose instead:**
- Set maxReplicas to 2× our current peak (e.g., 50 if we peak at 25)
- Add a billing alert at 150% of normal cost
- Add a namespace ResourceQuota so no single deployment can consume unbounded resources
- Set up a proper load test in staging to understand our actual breaking point"

</details>

---

### Q14. A junior engineer asks: "Should I always use HPA? It seems like it solves everything." What do you tell them?

<details>
<summary>💡 Senior Answer</summary>

"HPA is a great tool, but it's a tool for one specific problem: *stateless apps where the bottleneck is in the app itself and correlates with CPU/memory.*

Before adding HPA, I always ask:
1. Is my app actually stateless? If not, horizontal scaling is complicated.
2. Do my pods have `resources.requests` defined? Without this, HPA can't calculate utilization — it silently does nothing.
3. Is CPU/memory actually my bottleneck, or is it DB? Network? An external API?
4. Does my app start fast enough for reactive scaling to help?
5. What happens to downstream services (DB, cache) when I suddenly add 10× the pods?

HPA is often the last tool I reach for. My actual order is:
1. Profile the app — find the real bottleneck
2. Fix the bottleneck (better query, caching, async processing)
3. Set proper resource requests/limits
4. Set a sensible `minReplicas` based on baseline traffic
5. *Then* add HPA as a safety net for traffic spikes

Engineers who jump straight to 'add HPA' are often treating the symptom. The app is slow because of a missing database index, not because it needs 10× more pods."

</details>

---

## 📋 Quick Reference — HPA Diagnosis Commands

```bash
# Why is HPA not scaling?
kubectl describe hpa <name> -n <namespace>
# Look for: "FailedGetResourceMetric", "DesiredReplicas", "Conditions"

# Are pods actually sending metrics?
kubectl top pods -n <namespace>
# If this fails → metrics-server not installed or broken

# What resource requests do pods have?
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources.requests}{"\n"}{end}'

# HPA event history
kubectl get events -n <namespace> --field-selector involvedObject.name=<hpa-name>

# Live watch HPA decisions
kubectl get hpa -n <namespace> -w

# Check metrics server is working
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq .
```

---

## 🏆 Scoring Yourself

| Score | What it means |
|---|---|
| Can answer Q1–Q5 | You understand HPA well enough to use it safely |
| Can answer Q6–Q9 | You've either been in production or studied it seriously |
| Can answer Q10–Q12 | You think in systems, not just components |
| Can answer Q13–Q14 | You're ready to mentor others and push back on bad decisions |

> **Remember:** The goal isn't to have all the answers memorized.
> The goal is to know what questions to ask when you don't know the answer.
> Senior engineers say "let me check the metrics first" — not "I think it's X."
