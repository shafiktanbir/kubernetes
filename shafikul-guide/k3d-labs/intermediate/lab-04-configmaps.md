# 🟡 Lab 04 — Same App, Different Config Per Environment

**Level:** Intermediate
**Time:** ~30 minutes

---

## 🎬 Scenario

Your company runs the same webapp in 3 environments:
- `development` — debug mode ON, log level: DEBUG
- `staging` — debug mode OFF, log level: INFO
- `production` — debug mode OFF, log level: WARNING, max connections: 100

The team currently does this:

```bash
# ❌ What they do now (terrible practice)
image: myapp:v1 --debug=true --loglevel=DEBUG --max-conn=10
image: myapp:v1 --debug=false --loglevel=INFO --max-conn=50
image: myapp:v1 --debug=false --loglevel=WARNING --max-conn=100
```

They want you to make the image identical across all environments. **Config should come from K8s, not baked into the image.**

Your task: Use **ConfigMaps** to inject environment-specific config as environment variables.

---

## 📋 Setup

```bash
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```

**The app reads these environment variables:**
- `APP_DEBUG` → "true" or "false"
- `APP_LOG_LEVEL` → "DEBUG", "INFO", or "WARNING"
- `APP_MAX_CONNECTIONS` → number as string

**Try to solve it yourself first. Don't read below yet.**

---

## 💡 Hints

<details>
<summary>Hint 1 — ConfigMap basics</summary>

```bash
kubectl create configmap --help
# ConfigMaps store key-value pairs
# They can be injected as env vars or mounted as files
```
</details>

<details>
<summary>Hint 2 — How to use ConfigMap in a deployment</summary>

In your container spec, instead of hardcoding `env`, use:
```yaml
envFrom:
- configMapRef:
    name: <your-configmap-name>
```
</details>

<details>
<summary>Hint 3 — Verify env vars inside a pod</summary>

```bash
kubectl exec -it <pod-name> -n <namespace> -- env | grep APP_
```
</details>

---

## ✅ Full Solution

### Step 1: Create ConfigMaps Per Environment

```yaml
# dev-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  APP_DEBUG: "true"
  APP_LOG_LEVEL: "DEBUG"
  APP_MAX_CONNECTIONS: "10"
---
# staging-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: staging
data:
  APP_DEBUG: "false"
  APP_LOG_LEVEL: "INFO"
  APP_MAX_CONNECTIONS: "50"
---
# prod-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_DEBUG: "false"
  APP_LOG_LEVEL: "WARNING"
  APP_MAX_CONNECTIONS: "100"
```

```bash
kubectl apply -f dev-config.yaml
kubectl apply -f staging-config.yaml
kubectl apply -f prod-config.yaml
```

### Step 2: One Deployment YAML Used in All Environments

```yaml
# deployment.yaml — IDENTICAL for all environments
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production    # Change this per environment
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
        image: nginx:1.25    # Same image everywhere
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: app-config   # Pulls from the namespace's ConfigMap
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

```bash
# Deploy to all namespaces
kubectl apply -f deployment.yaml -n development
kubectl apply -f deployment.yaml -n staging
kubectl apply -f deployment.yaml -n production
```

### Step 3: Verify Each Environment Has Different Config

```bash
# Check dev pod
DEV_POD=$(kubectl get pod -n development -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $DEV_POD -n development -- env | grep APP_
# APP_DEBUG=true
# APP_LOG_LEVEL=DEBUG
# APP_MAX_CONNECTIONS=10

# Check prod pod
PROD_POD=$(kubectl get pod -n production -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $PROD_POD -n production -- env | grep APP_
# APP_DEBUG=false
# APP_LOG_LEVEL=WARNING
# APP_MAX_CONNECTIONS=100
```

### Step 4: Update Config Without Redeploying

```bash
# Change log level in production (e.g., temporarily enable INFO for debugging)
kubectl edit configmap app-config -n production
# Change APP_LOG_LEVEL to INFO
# Save and exit

# Pods need to restart to pick up new config
kubectl rollout restart deployment/webapp -n production

# Verify
kubectl exec $PROD_POD -n production -- env | grep APP_LOG_LEVEL
```

---

## 🔍 What You Learned

| Concept | Key Point |
|---------|-----------|
| ConfigMap | Stores non-sensitive config as key-value |
| `envFrom` | Injects all ConfigMap keys as env vars |
| Namespace isolation | Same ConfigMap name, different data per namespace |
| Live config update | Edit ConfigMap → restart pods to apply |

---

## ⚠️ Important: ConfigMap vs Secret

```
ConfigMap  → Non-sensitive data (log levels, feature flags, URLs)
Secret     → Sensitive data (passwords, API keys, tokens)
```
**Never put passwords in ConfigMaps.** That's Lab 05.

---

## 🔥 Bonus Challenges

**1. Mount ConfigMap as a file instead of env vars:**
```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
volumeMounts:
- name: config-volume
  mountPath: /etc/config
```
```bash
kubectl exec <pod> -- cat /etc/config/APP_LOG_LEVEL
```

**2. What happens if you delete a ConfigMap a running pod depends on?**
```bash
kubectl delete configmap app-config -n development
# Does the pod crash? Does it keep running?
```

```bash
# Cleanup
kubectl delete namespace development staging production
```
