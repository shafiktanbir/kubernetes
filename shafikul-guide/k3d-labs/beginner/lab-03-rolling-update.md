# 🟢 Lab 03 — Deploy New Version Without Downtime

**Level:** Beginner
**Time:** ~25 minutes

---




## 🎬 Scenario

Your team just shipped v2 of the webapp. You need to deploy it to production.

Your manager says:

> *"We cannot have any downtime. Users are active right now. If anything goes wrong with v2, you must be able to roll back to v1 within 60 seconds."*

The current deployment runs `nginx:1.24` (v1). You need to upgrade to `nginx:1.25` (v2).

---

## 📋 Setup

```bash
kubectl create namespace production

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra pod during update
      maxUnavailable: 1    # Allow 1 pod down at a time
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
        image: nginx:1.24    # Current version (v1)
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

Verify v1 is running:
```bash
kubectl get pods -n production
kubectl describe deployment webapp -n production | grep Image
# Image: nginx:1.24
```

**Now deploy v2 without downtime. Don't read below yet.**

---

## 💡 Hints

<details>
<summary>Hint 1 — How to update image</summary>

You can update the image directly without editing YAML:
```bash
kubectl set image --help
```
</details>

<details>
<summary>Hint 2 — Watch the rolling update live</summary>

Open a second terminal and run:
```bash
kubectl get pods -n production -w
```
You'll see old pods terminate one by one as new ones start.
</details>

<details>
<summary>Hint 3 — How to rollback</summary>

K8s keeps rollout history. Look at:
```bash
kubectl rollout --help
kubectl rollout history deployment webapp -n production
```
</details>

---

## ✅ Full Solution

### Part 1: Deploy v2 (Rolling Update)

```bash
# Update the image — K8s handles rolling update automatically
kubectl set image deployment/webapp webapp=nginx:1.25 -n production

# Watch the rolling update in real time (open second terminal)
kubectl rollout status deployment/webapp -n production

# You'll see:
# Waiting for deployment "webapp" rollout to finish: 1 out of 4 new replicas updated...
# Waiting for deployment "webapp" rollout to finish: 2 out of 4 new replicas updated...
# Waiting for deployment "webapp" rollout to finish: 3 out of 4 new replicas updated...
# deployment "webapp" successfully rolled out

# Verify new version
kubectl describe deployment webapp -n production | grep Image
# Image: nginx:1.25
```

### Part 2: Something Goes Wrong — Roll Back!

Simulate a bad deploy (wrong image):
```bash
kubectl set image deployment/webapp webapp=nginx:BROKEN -n production

# Watch pods fail
kubectl get pods -n production -w
# You'll see: ErrImagePull, ImagePullBackOff

# Check rollout status
kubectl rollout status deployment/webapp -n production
# Waiting for deployment... (stuck)
```

**Now roll back to the last working version:**
```bash
# See history
kubectl rollout history deployment/webapp -n production

# Roll back to previous version
kubectl rollout undo deployment/webapp -n production

# Watch recovery
kubectl rollout status deployment/webapp -n production

# Verify — should be back to nginx:1.25
kubectl describe deployment webapp -n production | grep Image
```

### Part 3: Roll Back to Specific Version

```bash
# See full history with details
kubectl rollout history deployment/webapp -n production --revision=1

# Roll back to revision 1 (nginx:1.24)
kubectl rollout undo deployment/webapp -n production --to-revision=1

# Verify
kubectl describe deployment webapp -n production | grep Image
# Image: nginx:1.24
```

---

## 🔍 What You Learned

| Concept | Command |
|---------|---------|
| Update image | `kubectl set image deployment/<name> <container>=<image>` |
| Watch rollout | `kubectl rollout status deployment/<name>` |
| See history | `kubectl rollout history deployment/<name>` |
| Roll back | `kubectl rollout undo deployment/<name>` |
| Roll back to specific version | `kubectl rollout undo --to-revision=<N>` |

---

## 🔥 Bonus Challenges

**1. Pause a rollout midway:**
```bash
kubectl set image deployment/webapp webapp=nginx:1.25 -n production
kubectl rollout pause deployment/webapp -n production
kubectl get pods -n production    # half old, half new
kubectl rollout resume deployment/webapp -n production
```

**2. How many revisions does K8s keep by default?**
```yaml
# Add this to your deployment spec:
spec:
  revisionHistoryLimit: 10   # default is 10
```

```bash
# Cleanup
kubectl delete namespace production
```
