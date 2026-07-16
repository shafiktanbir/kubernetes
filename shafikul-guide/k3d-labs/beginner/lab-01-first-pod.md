# 🟢 Lab 01 — Your First Deployment (Something Is Broken)

**Level:** Beginner
**Time:** ~20 minutes

---

## 🎬 Scenario

You just joined a company. Your team lead says:

> *"We need to deploy our web app. Here's the YAML. Apply it and make sure it's running."*

They hand you this file. You apply it. **It doesn't work.** Pods are not running.
Your job: **find the problem and fix it.**

---

## 📋 The Broken YAML

Create a file called `broken-app.yaml` and paste this:

```yaml
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
        app: webap        # <-- look carefully
    spec:
      containers:
      - name: webapp
        image: nginxxx:latest   # <-- look carefully
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

Apply it:

```bash
kubectl apply -f broken-app.yaml
```

Now check what's happening:

```bash
kubectl get pods -n production
```

You'll see problems. **Find them. Fix them. Don't read below yet.**

---

## 💡 Hints (Read Only If Stuck)

<details>
<summary>Hint 1 — First problem</summary>

The namespace `production` doesn't exist yet. K8s won't create it automatically.

```bash
kubectl get namespaces
```
</details>

<details>
<summary>Hint 2 — Second problem</summary>

Look at the labels carefully.
- `selector.matchLabels.app: webapp`
- `template.metadata.labels.app: webap`

These must match exactly. One is missing a letter.
</details>

<details>
<summary>Hint 3 — Third problem</summary>

`nginxxx` is not a real Docker image. It's a typo.
```bash
kubectl describe pod <pod-name> -n production
# Look for: ErrImagePull or ImagePullBackOff
```
</details>

---

## ✅ Full Solution

**Step 1: Create the namespace**
```bash
kubectl create namespace production
```

**Step 2: Fix the YAML — here's the corrected version:**

```yaml
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
        app: webapp          # ✅ fixed: was "webap"
    spec:
      containers:
      - name: webapp
        image: nginx:1.25    # ✅ fixed: was "nginxxx:latest"
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

**Step 3: Apply and verify**
```bash
kubectl apply -f broken-app.yaml
kubectl get pods -n production

# Expected output:
# NAME                      READY   STATUS    RESTARTS   AGE
# webapp-7d9f8b-xk2p1      1/1     Running   0          30s
# webapp-7d9f8b-mn3q2      1/1     Running   0          30s
# webapp-7d9f8b-pq4r5      1/1     Running   0          30s
```

---

## 🔍 What You Learned

| Problem | How to Detect | How to Fix |
|---------|--------------|------------|
| Namespace missing | `kubectl apply` error | `kubectl create namespace` |
| Label mismatch | Pods never start, 0 replicas | Match `selector` and `template.labels` exactly |
| Wrong image name | `ErrImagePull` / `ImagePullBackOff` | `kubectl describe pod` → fix image name |

---

## 🔥 Bonus Challenge

1. Delete one pod manually — watch K8s create a new one automatically
2. Try `kubectl get pods -n production -w` (watch mode) while deleting
3. Intentionally break the label again — observe what happens to existing pods

```bash
# Cleanup when done
kubectl delete namespace production
```
