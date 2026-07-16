# ⭐ Security — RBAC, Secrets, NetworkPolicy

---

## Q1: What is RBAC and how does it work in Kubernetes?

**Senior Answer:**

RBAC stands for Role-Based Access Control. It's K8s's authorization system — it controls what authenticated users and service accounts are allowed to do with the API.

There are four core objects:

**Role** — defines a set of permissions (verbs on resources) scoped to a single namespace.
**ClusterRole** — same as Role but cluster-wide, applies to all namespaces.
**RoleBinding** — assigns a Role or ClusterRole to a subject (User, Group, or ServiceAccount) within a namespace.
**ClusterRoleBinding** — assigns a ClusterRole cluster-wide.

The mental model: RBAC is additive only. There's no "deny" verb. By default, subjects have zero permissions. You only grant what they need — principle of least privilege.

**Example I'd give in an interview:**
```yaml
# Junior dev can deploy to dev namespace only
kind: Role
metadata:
  namespace: development
  name: deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update"]
# No "delete", no other namespaces
```

**Gotcha:** A ClusterRoleBinding with the built-in `cluster-admin` ClusterRole gives God-mode access to everything. Teams accidentally give this to their CI/CD bots and wonder why a pipeline bug deleted their production database.

---

## Q2: What is a ServiceAccount and why is it important?

**Senior Answer:**

A ServiceAccount is an identity for processes running inside Pods — as opposed to human user accounts. When a Pod needs to call the Kubernetes API (e.g., ArgoCD reading deployments, Prometheus scraping metrics, your app listing its own pods), it authenticates as a ServiceAccount.

Every namespace has a `default` ServiceAccount. If you don't specify one, your Pod runs as `default`. In older K8s versions, `default` had broad permissions — a security risk. Best practice: create a dedicated ServiceAccount per application with only the permissions it needs.

**Real example:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-bot
  namespace: production
---
# Then bind only what ArgoCD needs
kind: Role
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]
```

**The token injection:** K8s automatically mounts the ServiceAccount token as a file inside every Pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Your app reads this to authenticate to the API. In K8s 1.24+, these are short-lived tokens (1 hour) rotated automatically — much more secure than the old non-expiring tokens.

---

## Q3: How would you prevent a developer from accidentally deploying to production?

**Senior Answer:**

Multiple layers — defense in depth:

**Layer 1: RBAC** — developers get a Role in `staging` and `development` namespaces only. No RoleBinding in `production`. `kubectl apply -f deployment.yaml -n production` returns 403 Forbidden.

**Layer 2: GitOps with ArgoCD** — no one runs kubectl in production. All changes go through a Git PR. Only ArgoCD (running as a ServiceAccount with deploy permissions) touches production. Developers can't bypass this even if they have `kubectl`.

**Layer 3: Admission Controllers / OPA Gatekeeper** — policy enforcement at the API server level. Example: "No image tag `latest` allowed in production namespace," "All deployments must have resource limits," "No privileged containers."

**Layer 4: Branch protection** — `main` branch requires 2 approvals before merge. ArgoCD syncs from `main`. So deploying to production requires both a code review AND ArgoCD sync — no single person can do it alone.

I'd say the right answer in most companies is Layer 1 + Layer 2. Layers 3 and 4 are extra hardening for regulated industries.

---

## Q4: How do you handle secrets securely in Kubernetes?

**Senior Answer:**

There are levels of sophistication here, and I'd present them honestly:

**Level 1 — Basic K8s Secrets:** Better than hardcoding in YAML or environment variables, but Secrets are only base64-encoded in etcd, not encrypted. Anyone with etcd access can read them. Acceptable for non-sensitive environments or small teams.

**Level 2 — Encryption at Rest:** Enable etcd encryption with a KMS provider (AWS KMS, GCP KMS). Now Secrets are genuinely encrypted at rest. The encryption key lives outside etcd. This is the standard for production at most companies.

**Level 3 — External Secret Managers:** HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager. Secrets never enter K8s etcd at all. The CSI Secrets Store driver or the External Secrets Operator injects secrets into pods at startup as mounted files or env vars. Full audit logs of every secret access. Automatic rotation.

**My recommendation:** For a greenfield project I'd start with Level 2 (encryption at rest enabled). For anything handling PII, payment data, or healthcare data — Level 3 with Vault or cloud KMS.

**What I'd never do:** Put secrets in ConfigMaps (they're not marked sensitive), hardcode them in Dockerfiles, or commit them to Git. There are GitHub Actions that scan for leaked secrets — one mistake and your credentials are public.

---

## Q5: What is a NetworkPolicy and why is it important?

**Senior Answer:**

By default, every Pod in a Kubernetes cluster can talk to every other Pod in any namespace. That's a massive security risk — if an attacker compromises your frontend pod, they have network access to your database pods.

A NetworkPolicy is a firewall rule for Pod-to-Pod traffic. It lets you define ingress (incoming) and egress (outgoing) rules based on pod selectors, namespace selectors, and IP blocks.

**Classic example:**
```yaml
# Only allow the API pod to reach the database pod
# Block everyone else
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-only-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database          # This policy applies to database pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api           # Only from API pods
    ports:
    - protocol: TCP
      port: 5432
```

**Critical gotcha:** NetworkPolicy requires a CNI plugin that supports it — Calico, Cilium, Weave. The default k3d CNI (Flannel) does NOT enforce NetworkPolicy. You'd need to install Calico or Cilium for policies to actually work. Many teams write NetworkPolicy YAMLs and think they're secure, but their CNI silently ignores them.

---

## Q6: What is Pod Security Admission (PSA)?

**Senior Answer:**

PSA replaced PodSecurityPolicies (PSP, deprecated in 1.21, removed in 1.25). It's a built-in admission controller that enforces security standards on pods at the namespace level.

There are three built-in policy levels:

**Privileged** — no restrictions. For system-level workloads like CNI plugins.
**Baseline** — prevents known privilege escalation (no privileged containers, no hostPath mounts). The minimum for production workloads.
**Restricted** — heavily hardened. Requires non-root, no privilege escalation, read-only root filesystem. Hard to adopt but ideal for sensitive workloads.

You apply it with namespace labels:
```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

Now if someone tries to deploy a container running as root in that namespace, the API server rejects it.

**In practice:** Most companies adopt `baseline` for production namespaces and run a migration project to get to `restricted`. It catches common misconfigurations — developers who run containers as root because "it was easier in development."
