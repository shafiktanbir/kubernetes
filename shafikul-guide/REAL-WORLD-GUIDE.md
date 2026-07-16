# 🚨 Real World Guide — After The Labs

> **Read this AFTER completing all 12 labs.**
> This guide exists for one reason: to prevent false confidence.
>
> The labs taught you concepts. This guide tells you what you actually need
> to build, break, and survive before you're genuinely job-ready.

---

## Honest Self-Assessment First

Before going further — answer these questions honestly:

```
Can you explain CrashLoopBackOff debug steps without looking at notes?   Y/N
Can you write a Deployment YAML from scratch in 5 minutes?               Y/N
Can you explain ArgoCD to someone who has never heard of it?             Y/N
Have you broken your cluster and recovered it without help?              Y/N
Have you deployed YOUR OWN app (not nginx) to K8s?                      Y/N
```

If any answer is **N** — go back to the labs. Don't move forward yet.

---

## The 5 Real-World Projects You Must Build

These are not optional. Each one closes a critical gap between "lab experience" and "production experience."

---

## Project 1: Deploy Your Own Real Backend App

**The gap:** Every lab used `nginx` as a toy. Companies deploy real applications.

**What to build:**

Pick ONE backend app you've already built (Node.js, Go, Python, anything) OR build a simple one:

```
Simple REST API with:
  - GET  /health       → 200 OK (for readiness probe)
  - GET  /users        → returns list from PostgreSQL
  - POST /users        → inserts into PostgreSQL
  - GET  /metrics      → Prometheus metrics endpoint
```

**Then do ALL of this:**

```bash
# Step 1: Write a proper multi-stage Dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.19
COPY --from=builder /app/server .
EXPOSE 8080
USER 1001                    # Run as non-root
CMD ["./server"]

# Step 2: Push to GitHub Container Registry
docker build -t ghcr.io/YOUR_USERNAME/myapp:v1.0.0 .
docker push ghcr.io/YOUR_USERNAME/myapp:v1.0.0

# Step 3: Write your own Deployment YAML (not from a tutorial)
# Add readiness probe that hits /health
# Add liveness probe
# Add resource requests and limits
# Connect to PostgreSQL via Secrets

# Step 4: Deploy to k3d
kubectl apply -f deployment.yaml
curl http://myapp.local/users   # should work
```

**You've succeeded when:**
- Your app is running in k3d
- Readiness probe is working (kill the DB and watch pod go NotReady)
- App reads DB_PASSWORD from a K8s Secret (not hardcoded)
- You can do a rolling update from v1.0.0 → v1.0.1 with zero downtime

**Time estimate:** 2-3 days

---

## Project 2: Full CI/CD Pipeline (GitHub Actions → ArgoCD)

**The gap:** Lab 10 shows ArgoCD. It doesn't show how code changes trigger it automatically.

**What to build:**

```
git push to main
      ↓
GitHub Actions:
  ├── run unit tests
  ├── build Docker image
  ├── tag image with git SHA (not "latest")
  ├── push to ghcr.io
  └── update image tag in helm/values.yaml
            ↓
          git commit + push
                ↓
            ArgoCD detects Git changed
                ↓
            ArgoCD syncs new image to cluster
                ↓
            Rolling update — zero downtime
```

**The GitHub Actions pipeline:**

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Run tests
      run: go test ./...

    - name: Build and push image
      run: |
        IMAGE=ghcr.io/${{ github.repository }}:${{ github.sha }}
        docker build -t $IMAGE .
        echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
        docker push $IMAGE

    - name: Update image tag in Helm values
      run: |
        sed -i "s|tag:.*|tag: ${{ github.sha }}|" helm/myapp/values.yaml
        git config user.email "ci@github.com"
        git config user.name "GitHub Actions"
        git commit -am "ci: deploy ${{ github.sha }}"
        git push
```

**You've succeeded when:**
- You push a code change to GitHub
- 3 minutes later, without touching kubectl, the new version is running in your cluster
- Every commit in Git history maps to exactly what ran in the cluster

**Time estimate:** 2-3 days

---

## Project 3: Add TLS with cert-manager

**The gap:** All your Ingress is HTTP. Every real production system uses HTTPS. Companies reject candidates who don't know how TLS works in K8s.

**What to build:**

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Create a ClusterIssuer (using Let's Encrypt staging first)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: traefik
EOF

# Update your Ingress to request a certificate
# cert-manager automatically provisions and renews it
```

**For local k3d testing** (no real domain needed):

```bash
# Use a self-signed certificate for local development
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
  namespace: production
spec:
  secretName: myapp-tls-secret
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
  - myapp.local
EOF
```

**You've succeeded when:**
- Your app runs on `https://myapp.local`
- cert-manager automatically created the TLS Secret
- You understand what ACME, ClusterIssuer, and Certificate objects are

**Time estimate:** 1 day

---

## Project 4: Simulate a Production Incident and Write a Postmortem

**The gap:** Labs show you how things break. Real experience means you've diagnosed something unexpected and written down what happened.

**The exercise:**

Break your cluster in 3 different ways, without looking at the solution first. For each one, document:

```markdown
## Incident Report: [Description]

**Date:** ...
**Duration:** ... minutes to resolve
**Impact:** ...

### What happened
...

### How I detected it
...

### Debug steps I took (in order)
1. I ran: `kubectl get pods -n production` — saw X
2. I ran: `kubectl describe pod Y` — saw Z
3. ...

### Root cause
...

### How I fixed it
...

### What I'd do differently
...
```

**Scenarios to break and diagnose:**

```bash
# Scenario A: Memory bomb
kubectl set resources deployment myapp --limits=memory=5Mi
# Your app OOMKills. Diagnose and fix without looking at notes.

# Scenario B: Config disappears
kubectl delete configmap app-config -n production
# Your app starts crashing. Why? Fix it.

# Scenario C: Wrong image deployed
kubectl set image deployment/myapp myapp=myapp:nonexistent-tag
# Cluster degraded. Some pods old, some failing. Fix with zero downtime.

# Scenario D: Node failure during high traffic
k3d node stop k3d-practice-agent-0
# Traffic spikes simultaneously (run load gen). Can cluster handle it?
# How long until recovery?
```

**You've succeeded when:**
- You can diagnose each scenario without hints in under 10 minutes
- You have 4 written postmortems documenting what you found

**Time estimate:** 3-4 days (space them out)

---

## Project 5: Multi-Service Architecture With Real Dependencies

**The gap:** Labs deploy services in isolation. Production systems have service-to-service calls, shared databases, and startup dependencies.

**What to build:**

```
┌─────────────────────────────────────────────────┐
│                K8s Cluster                      │
│                                                 │
│  Ingress (myapp.local)                          │
│       │                                         │
│       ├─── /         → frontend (nginx)         │
│       └─── /api      → api-service (your app)  │
│                            │                   │
│                            ├── postgres (StatefulSet)
│                            └── redis (StatefulSet)
│                                                 │
└─────────────────────────────────────────────────┘
```

**Requirements:**
- `api-service` reads/writes from PostgreSQL
- `api-service` caches responses in Redis
- DB credentials in K8s Secrets
- `initContainers` in api-service wait for PostgreSQL to be ready before starting
- `HorizontalPodAutoscaler` on api-service (min 2, max 10)
- Separate namespaces for `frontend` and `backend` workloads
- RBAC: create a ServiceAccount for api-service with minimal permissions

**The startup dependency pattern:**

```yaml
# initContainer waits for DB before main app starts
initContainers:
- name: wait-for-postgres
  image: busybox
  command: ['sh', '-c',
    'until nc -z postgres-service 5432; do echo waiting for postgres; sleep 2; done']
```

**You've succeeded when:**
- All 4 services are running and communicating
- You can kill PostgreSQL and api-service goes NotReady (not crash)
- You can do a rolling update of api-service without downtime

**Time estimate:** 4-5 days

---

## The Brutal Checklist

Do this checklist every 2 weeks. Be honest. Only check ✅ when you can do it without notes.

### Fundamentals
- [ ] Write a Deployment YAML from scratch in under 5 minutes
- [ ] Debug CrashLoopBackOff without any hints in under 10 minutes
- [ ] Explain RBAC to a non-technical person clearly
- [ ] Describe what happens step by step when you run `kubectl apply`

### Hands-On
- [ ] Deployed MY OWN real application to K8s (not nginx)
- [ ] Built a full CI/CD pipeline (code push → auto deploy)
- [ ] Set up TLS with cert-manager
- [ ] Configured HPA and watched it actually scale under load
- [ ] Simulated node failure and recovered without help
- [ ] Written at least 2 incident postmortems

### Production Readiness
- [ ] Can set up Prometheus + Grafana from scratch without a tutorial
- [ ] Know how to check if NetworkPolicy is actually being enforced
- [ ] Can explain when NOT to use Kubernetes
- [ ] Understand what etcd is and why it matters
- [ ] Can do a K8s version upgrade (even on local cluster)

### Interview Ready
- [ ] Can explain ArgoCD vs GitHub Actions without notes
- [ ] Can whiteboard the K8s control plane from memory
- [ ] Can answer "how does DNS work in K8s" in 2 minutes
- [ ] Have a real story for "tell me about a production incident you debugged"

---

## Warning Signs You're Not Ready Yet

```
❌ You can do the labs but can't do them on a fresh cluster without notes
❌ You've never deployed code you actually wrote
❌ Your debugging always starts with Google instead of kubectl
❌ You can't explain a concept in plain English to someone non-technical
❌ You've never written a Dockerfile from scratch
❌ You feel ready after just the 12 labs
```

---

## Realistic Timeline

```
Month 1:  Complete all 12 labs + interview prep reading
Month 2:  Project 1 (real app) + Project 3 (TLS)
Month 3:  Project 2 (CI/CD pipeline) + Project 4 (incidents)
Month 4:  Project 5 (multi-service) + start applying to jobs
Month 5+: On the job — this is where senior-level experience comes from
```

> The engineers who land senior DevOps/SRE roles didn't just study K8s.
> They broke K8s. Repeatedly. At 2am. And fixed it.
> That's what you're building toward.
