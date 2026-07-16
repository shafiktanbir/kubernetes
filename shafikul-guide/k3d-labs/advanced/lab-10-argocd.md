# 🔴 Lab 10 — Deploy with ArgoCD (GitOps Style)

**Level:** Advanced
**Time:** ~50 minutes

---

## 🎬 Scenario

Your team is tired of manual deployments. Every deploy is:
- Someone SSHing into the server
- Running `kubectl apply` manually
- Nobody knows who deployed what
- Staging and production drifting from each other

Your manager says:

> *"I want every deployment to go through Git. No more manual kubectl. If you want to deploy, you commit to Git. ArgoCD does the rest. If the cluster doesn't match Git, ArgoCD should fix it automatically."*

**Your task:** Install ArgoCD, connect it to a Git repo, and deploy your app via GitOps.

---

## 📋 Prerequisites

- k3d cluster running (from setup)
- A GitHub account (free)

---

## 💡 What You're Building

```
You commit YAML to GitHub
         ↓
ArgoCD detects the change (polls every 3 min)
         ↓
ArgoCD applies the diff to your cluster
         ↓
Your app is updated — no manual kubectl needed
```

---

## ✅ Full Solution

### Step 1: Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready (takes 2-3 minutes)
kubectl wait --for=condition=Ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd \
  --timeout=300s

# Verify all pods are running
kubectl get pods -n argocd
```

### Step 2: Access ArgoCD UI

```bash
# Expose ArgoCD server (port-forward for local access)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
# Copy this password

# Open browser: https://localhost:8080
# Username: admin
# Password: (copied above)
# Accept the self-signed certificate warning
```

### Step 3: Set Up Your Git Repository

Create a new **public** GitHub repository called `k8s-gitops-demo`.

In that repo, create this file structure:

```
k8s-gitops-demo/
└── apps/
    └── webapp/
        ├── namespace.yaml
        ├── deployment.yaml
        └── service.yaml
```

**namespace.yaml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

**deployment.yaml:**
```yaml
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
        image: nginx:1.24       # Start with 1.24 — we'll upgrade via Git
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

**service.yaml:**
```yaml
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
  type: ClusterIP
```

Commit and push these files to GitHub.

### Step 4: Connect ArgoCD to Your GitHub Repo

**Option A: Via UI**
1. Open https://localhost:8080
2. Click **"+ New App"**
3. Fill in:
   - **Application Name:** `webapp`
   - **Project:** `default`
   - **Sync Policy:** `Automatic`
   - **Repository URL:** `https://github.com/YOUR_USERNAME/k8s-gitops-demo`
   - **Path:** `apps/webapp`
   - **Cluster URL:** `https://kubernetes.default.svc`
   - **Namespace:** `production`
4. Click **Create**

**Option B: Via YAML (preferred — reproducible)**

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/k8s-gitops-demo
    targetRevision: HEAD
    path: apps/webapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Fix cluster drift automatically
    syncOptions:
    - CreateNamespace=true
```

```bash
# Replace YOUR_USERNAME first!
kubectl apply -f argocd-app.yaml

# Check app status
kubectl get application webapp -n argocd
```

### Step 5: Verify GitOps is Working

```bash
# App should sync and deploy your webapp
kubectl get pods -n production
# NAME                      READY   STATUS    RESTARTS
# webapp-xxx-yyy            1/1     Running   0

# Check ArgoCD sync status
kubectl get application webapp -n argocd -o yaml | grep -A5 "status:"
```

### Step 6: Deploy via Git (The Whole Point)

```bash
# Change nginx:1.24 to nginx:1.25 in deployment.yaml on GitHub
# Commit: "feat: upgrade nginx to 1.25"
# Push to GitHub

# Wait 3 minutes (ArgoCD poll interval) OR manually sync:
kubectl -n argocd patch application webapp \
  -p '{"operation": {"initiatedBy": {"username": "shafikul"}, "sync": {}}}' \
  --type merge

# Watch the rollout
kubectl rollout status deployment/webapp -n production

# Verify new image
kubectl describe deployment webapp -n production | grep Image
# Image: nginx:1.25
```

### Step 7: Test Self-Healing (Core ArgoCD Feature)

```bash
# Manually scale to 10 replicas (simulating someone bypassing Git)
kubectl scale deployment webapp -n production --replicas=10

# Watch what happens
kubectl get pods -n production -w

# ArgoCD detects drift within 3 minutes
# Automatically reverts back to 2 replicas (what Git says)
kubectl get pods -n production
# Only 2 pods — ArgoCD corrected it
```

---

## 🔍 ArgoCD App States

```
Synced    → Cluster matches Git ✅
OutOfSync → Cluster drifts from Git ⚠️
Healthy   → All pods running fine ✅
Degraded  → Some pods failing ❌
Progressing → Deployment in progress 🔄
```

```bash
# Check status from CLI
kubectl get application webapp -n argocd
# NAME     SYNC STATUS   HEALTH STATUS
# webapp   Synced        Healthy
```

---

## 🔥 Bonus Challenges

**1. Intentionally break the deployment and watch ArgoCD fix it:**
```bash
kubectl delete deployment webapp -n production
# ArgoCD recreates it within minutes (selfHeal: true)
```

**2. Disable auto-sync temporarily (for maintenance):**
```bash
argocd app set webapp --sync-policy none
# Now it won't auto-sync until you re-enable
argocd app set webapp --sync-policy automated
```

**3. View GitOps history in ArgoCD UI:**
- Every sync is tied to a Git commit
- Click "History and Rollback" in UI
- Roll back to any previous Git commit with one click

```bash
# Cleanup ArgoCD app (keeps argocd installed for future labs)
kubectl delete application webapp -n argocd
kubectl delete namespace production
```
