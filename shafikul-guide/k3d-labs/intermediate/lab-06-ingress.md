# 🟡 Lab 06 — Expose Your App to the Outside World

**Level:** Intermediate
**Time:** ~35 minutes

---

## 🎬 Scenario

Your app is running in the cluster. Your manager asks:

> *"Why can't I access the app from my browser? I go to http://myapp.local and get nothing."*

You realize the app is only accessible inside the cluster. You need to expose it externally using **Ingress** with proper routing.

**Requirements:**
- `http://myapp.local/` → webapp (nginx)
- `http://myapp.local/api` → api-service
- Both run as separate deployments

---

## 📋 Setup

```bash
kubectl create namespace production

# Deploy the frontend (webapp)
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

# Deploy the API
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:latest
        args:
        - "-text=Hello from API v1"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678
EOF
```

Both apps run but are not reachable from outside. **Your task: create the Ingress.**

---

## 💡 Hints

<details>
<summary>Hint 1 — Understanding Services vs Ingress</summary>

```
ClusterIP  → Only inside cluster (default)
NodePort   → Exposes on node's IP:port (not for production)
Ingress    → HTTP/HTTPS routing by hostname/path (production way)
```
</details>

<details>
<summary>Hint 2 — k3d comes with Traefik Ingress controller built in</summary>

```bash
kubectl get pods -n kube-system | grep traefik
# traefik-xxxxx   Running   ← It's already there
```
</details>

<details>
<summary>Hint 3 — Add local hostname</summary>

You need to add `myapp.local` to your `/etc/hosts` to test locally:
```bash
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts
```
</details>

---

## ✅ Full Solution

### Step 1: Understand Current State

```bash
# Services exist but are ClusterIP (internal only)
kubectl get services -n production
# NAME             TYPE        CLUSTER-IP     PORT(S)
# webapp-service   ClusterIP   10.43.x.x      80/TCP
# api-service      ClusterIP   10.43.x.x      80/TCP
```

### Step 2: Add Local Hostname

```bash
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts
```

### Step 3: Create the Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: production
  annotations:
    # Traefik specific (k3d uses Traefik by default)
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml

# Verify ingress was created
kubectl get ingress -n production
# NAME             CLASS    HOSTS        ADDRESS     PORTS
# webapp-ingress   traefik  myapp.local  172.x.x.x   80
```

### Step 4: Test It

```bash
# Test frontend
curl http://myapp.local
# Should return nginx default page HTML

# Test API route
curl http://myapp.local/api
# Should return: Hello from API v1

# Or open in browser
# http://myapp.local     → nginx page
# http://myapp.local/api → Hello from API v1
```

---

## 🔍 How Traffic Flows

```
Your Browser
    │
    │ http://myapp.local (port 80)
    ▼
k3d LoadBalancer (port 80 mapped to your PC)
    │
    ▼
Traefik Ingress Controller (pod in kube-system)
    │
    ├── path /api  → api-service → api pods
    └── path /     → webapp-service → webapp pods
```

---

## 🔥 Bonus Challenges

**1. Add a second hostname:**
```yaml
# Add to ingress rules:
- host: api.myapp.local
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: api-service
          port:
            number: 80
```
```bash
echo "127.0.0.1 api.myapp.local" | sudo tee -a /etc/hosts
curl http://api.myapp.local
```

**2. What happens if you hit a path that doesn't exist?**
```bash
curl http://myapp.local/nonexistent
# What HTTP status do you get?
```

**3. Rate limiting with annotations (Traefik):**
```yaml
annotations:
  traefik.ingress.kubernetes.io/router.middlewares: default-ratelimit@kubernetescrd
```

```bash
# Cleanup
kubectl delete namespace production
# Remove from /etc/hosts manually or:
sudo sed -i '/myapp.local/d' /etc/hosts
```
