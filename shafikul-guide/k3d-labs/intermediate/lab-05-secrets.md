# 🟡 Lab 05 — DB Password Must Not Be in Git

**Level:** Intermediate
**Time:** ~30 minutes

---

## 🎬 Scenario

You're doing a code review. A junior dev submitted this pull request:

```yaml
# ❌ junior-dev's deployment.yaml (in Git)
env:
- name: DB_PASSWORD
  value: "SuperSecret123!"
- name: AWS_SECRET_KEY
  value: "AKIAIOSFODNN7EXAMPLE"
- name: STRIPE_API_KEY
  value: "sk_live_abc123xyz789"
```

You reject the PR and explain: **Secrets must NEVER be in Git.**

Your task: Move these credentials into **Kubernetes Secrets** and inject them safely.

---

## 📋 Setup

```bash
kubectl create namespace production
```

The app needs these environment variables:
- `DB_HOST` = `postgres.production.svc.cluster.local`
- `DB_PASSWORD` = `SuperSecret123!`
- `DB_USER` = `appuser`
- `STRIPE_API_KEY` = `sk_live_abc123xyz789`

**Try to solve it yourself first.**

---

## 💡 Hints

<details>
<summary>Hint 1 — Creating a secret</summary>

```bash
kubectl create secret generic --help
# Secrets are base64 encoded (not encrypted by default in k3d)
# But they are NOT in Git — that's the key difference
```
</details>

<details>
<summary>Hint 2 — Inspecting a secret</summary>

```bash
kubectl get secret <name> -n production -o yaml
# Values are base64 encoded
# Decode: echo "base64value" | base64 -d
```
</details>

<details>
<summary>Hint 3 — Injecting secret into pod</summary>

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: <secret-name>
      key: <key-name>
```
</details>

---

## ✅ Full Solution

### Step 1: Create the Secrets (Never Commit These Commands to Git)

```bash
# Create DB credentials secret
kubectl create secret generic db-credentials \
  --from-literal=DB_PASSWORD=SuperSecret123! \
  --from-literal=DB_USER=appuser \
  -n production

# Create payment credentials secret
kubectl create secret generic payment-credentials \
  --from-literal=STRIPE_API_KEY=sk_live_abc123xyz789 \
  -n production

# Verify (values will be base64 encoded)
kubectl get secrets -n production
```

### Step 2: Inspect the Secret (Educational)

```bash
kubectl get secret db-credentials -n production -o yaml
```

Output:
```yaml
apiVersion: v1
kind: Secret
type: Opaque
data:
  DB_PASSWORD: U3VwZXJTZWNyZXQxMjMh    # base64 encoded
  DB_USER: YXBwdXNlcg==
```

```bash
# Decode it (proves it's just base64, not encrypted)
echo "U3VwZXJTZWNyZXQxMjMh" | base64 -d
# SuperSecret123!
```

> ⚠️ **Important**: K8s Secrets are only base64, not encrypted. In production,
> companies add encryption at rest or use HashiCorp Vault. But it's still
> 1000x better than hardcoding in YAML.

### Step 3: Deployment That Reads From Secrets

```yaml
# safe-deployment.yaml
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
        env:
        # Non-sensitive config directly (fine to do)
        - name: DB_HOST
          value: "postgres.production.svc.cluster.local"
        # Sensitive values — pulled from Secrets
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: payment-credentials
              key: STRIPE_API_KEY
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

```bash
kubectl apply -f safe-deployment.yaml

# Verify secrets are injected as env vars inside the pod
POD=$(kubectl get pod -n production -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -n production -- env | grep -E "DB_|STRIPE"
# DB_HOST=postgres.production.svc.cluster.local
# DB_USER=appuser
# DB_PASSWORD=SuperSecret123!
# STRIPE_API_KEY=sk_live_abc123xyz789
```

### Step 4: What Happens If Secret Is Missing?

```bash
# Delete the secret
kubectl delete secret payment-credentials -n production

# Try to deploy a new pod (rollout restart forces new pod)
kubectl rollout restart deployment/webapp -n production

# Watch what happens
kubectl get pods -n production
# STATUS: CreateContainerConfigError

kubectl describe pod <new-pod-name> -n production
# Error: secret "payment-credentials" not found
```

This is a **fail-safe** — the pod refuses to start if a required secret doesn't exist.

---

## 🔍 What You Learned

| Concept | Key Point |
|---------|-----------|
| K8s Secret | Stores sensitive data, base64 encoded, not in Git |
| `secretKeyRef` | Injects specific secret key as env var |
| `envFrom secretRef` | Injects ALL keys from secret as env vars |
| Missing secret | Pod fails with `CreateContainerConfigError` |
| Base64 ≠ Encryption | Secrets need encryption at rest for true security |

---

## 🔥 Bonus Challenges

**1. Inject all secrets at once with `envFrom`:**
```yaml
envFrom:
- secretRef:
    name: db-credentials
- secretRef:
    name: payment-credentials
```

**2. Mount secret as a file (useful for TLS certs, SSH keys):**
```yaml
volumes:
- name: secret-volume
  secret:
    secretName: db-credentials
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
  readOnly: true
```
```bash
kubectl exec <pod> -n production -- cat /etc/secrets/DB_PASSWORD
```

**3. Update a secret without redeploying:**
```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_PASSWORD=NewPassword456! \
  --from-literal=DB_USER=appuser \
  -n production \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment/webapp -n production
```

```bash
# Cleanup
kubectl delete namespace production
```
