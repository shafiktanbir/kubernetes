# 🟡 Lab 07 — Intern Shouldn't Touch Production

**Level:** Intermediate
**Time:** ~35 minutes

---

## 🎬 Scenario

Your company just hired 2 new people:

- **Alice** — junior frontend developer, should only deploy to `dev` namespace
- **Bob** — QA engineer, needs to read logs and exec into pods but cannot change anything

Your team lead says:

> *"Last week someone accidentally deleted the production deployment. We need proper access controls NOW. Set up RBAC."*

Currently everyone has full cluster access. **Fix it.**

---

## 📋 Setup

```bash
kubectl create namespace development
kubectl create namespace production

# Deploy something to work with
kubectl create deployment webapp --image=nginx:1.25 -n development
kubectl create deployment webapp --image=nginx:1.25 -n production
```

**Don't read below yet — try to figure out Role, ClusterRole, RoleBinding, ClusterRoleBinding.**

---

## 💡 Hints

<details>
<summary>Hint 1 — RBAC building blocks</summary>

```
Role           → permissions scoped to ONE namespace
ClusterRole    → permissions across ALL namespaces
RoleBinding    → assigns Role or ClusterRole to a user in a namespace
ClusterRoleBinding → assigns ClusterRole cluster-wide
```
</details>

<details>
<summary>Hint 2 — What verbs exist?</summary>

```
get, list, watch         → read operations
create, update, patch    → write operations
delete                   → dangerous
```
</details>

<details>
<summary>Hint 3 — Test without real users</summary>

```bash
# Impersonate a user to test their permissions
kubectl auth can-i <verb> <resource> --as=<username> -n <namespace>
kubectl auth can-i delete deployments --as=alice -n production
```
</details>

---

## ✅ Full Solution

### Alice's Permissions: Deploy to dev only

```yaml
# alice-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development       # Scoped to dev ONLY
rules:
# Can manage deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# Can read pods (to see her deployments)
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
# Can manage services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# Cannot delete (safety net)
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: development
subjects:
- kind: User
  name: alice               # This must match the user's cert/OIDC name
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### Bob's Permissions: Read-only across namespaces

```yaml
# bob-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole           # ClusterRole = all namespaces
metadata:
  name: readonly-debugger
rules:
# Read pods and logs
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch", "create"]   # create for exec
# Read deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
# Read events (useful for debugging)
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
# Read services and configmaps (but NOT secrets)
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding     # Applies cluster-wide
metadata:
  name: bob-readonly
subjects:
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: readonly-debugger
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f alice-role.yaml
kubectl apply -f bob-role.yaml
```

### Test the Permissions

```bash
# ✅ Alice CAN deploy to dev
kubectl auth can-i create deployments --as=alice -n development
# yes

# ❌ Alice CANNOT touch production
kubectl auth can-i create deployments --as=alice -n production
# no

# ❌ Alice CANNOT delete in dev
kubectl auth can-i delete deployments --as=alice -n development
# no

# ✅ Bob CAN read pods in production
kubectl auth can-i get pods --as=bob -n production
# yes

# ✅ Bob CAN exec into pods (for debugging)
kubectl auth can-i create pods/exec --as=bob -n production
# yes

# ❌ Bob CANNOT delete anything
kubectl auth can-i delete deployments --as=bob -n production
# no

# ❌ Bob CANNOT read secrets
kubectl auth can-i get secrets --as=bob -n production
# no
```

### Protect Production From Everyone Except Senior Devs

```yaml
# prod-deployer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prod-deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]   # update only — no create/delete
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: senior-dev-prod
  namespace: production
subjects:
- kind: Group
  name: senior-devs            # Group of users
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: prod-deployer
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔍 Decision Guide — What To Use When

```
Question: Does this person need access to ONE namespace?
  YES → Use Role + RoleBinding

Question: Does this person need READ access across ALL namespaces?
  YES → Use ClusterRole + ClusterRoleBinding (read-only)

Question: Should anyone be able to DELETE in production?
  NO  → Don't include "delete" verb in any production Role

Question: Should CI/CD bots have access?
  YES → Create a ServiceAccount, not a User
```

---

## 🔥 Bonus Challenges

**1. Create a ServiceAccount for a CI/CD bot:**
```bash
kubectl create serviceaccount ci-bot -n production
# Then create Role + RoleBinding for ci-bot ServiceAccount
```

**2. What built-in ClusterRoles exist?**
```bash
kubectl get clusterroles | head -30
# view, edit, admin, cluster-admin are common ones
```

**3. Use the built-in `view` ClusterRole for Bob (shortcut):**
```bash
kubectl create clusterrolebinding bob-view \
  --clusterrole=view \
  --user=bob
```

```bash
# Cleanup
kubectl delete namespace development production
kubectl delete clusterrolebinding bob-readonly bob-view 2>/dev/null
kubectl delete clusterrole readonly-debugger 2>/dev/null
```
