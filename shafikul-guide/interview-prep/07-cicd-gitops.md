# ⭐⭐ CI/CD & GitOps — ArgoCD, Helm, Pipelines

---

## Q1: What problem does ArgoCD solve that GitHub Actions cannot?

**Senior Answer:**

GitHub Actions is a push-based CI/CD tool — it fires on an event (push, PR merge), runs commands, and then stops. It has no awareness of what happens in the cluster after deployment. If someone manually changes the cluster, GitHub Actions doesn't know and doesn't care.

ArgoCD is a pull-based GitOps operator that runs permanently inside the cluster. It continuously reconciles the cluster state against a Git repository. If anything drifts — someone manually scales a deployment, deletes a ConfigMap, changes an image tag — ArgoCD detects the drift within minutes and corrects it automatically.

**The core problems ArgoCD solves that GitHub Actions can't:**

1. **Configuration drift** — `kubectl scale deployment --replicas=20` at 3am → ArgoCD reverts it back to what Git says within 3 minutes. GitHub Actions: no idea.

2. **Security** — ArgoCD runs inside the cluster and pulls from Git. Your cluster never needs to expose an inbound port to GitHub. GitHub Actions needs your cluster credentials stored as GitHub Secrets and makes outbound calls to your cluster — a larger attack surface.

3. **Audit trail** — Every sync in ArgoCD is tied to a Git commit SHA + author. Rollback = `git revert`. GitHub Actions logs can be deleted; Git history is permanent.

4. **Multi-cluster** — One ArgoCD instance can manage 50 clusters. GitHub Actions would need 50 copies of your pipeline with credentials for each.

**How I'd use both together:** GitHub Actions handles CI — build, test, push image, update the image tag in the Git manifest. ArgoCD handles CD — detects the Git change and syncs to the cluster. Clear separation of concerns.

---

## Q2: What is GitOps and why is it better than traditional deployment pipelines?

**Senior Answer:**

GitOps is an operational model where Git is the single source of truth for your system's desired state. You declare what the cluster should look like in Git. An automated agent continuously ensures the cluster matches that declaration.

**Traditional pipeline:**
```
Developer → pushes code → pipeline runs kubectl apply → done
```
Problems:
- Cluster state exists only in the cluster, not reproducibly in code
- Two engineers can deploy conflicting changes simultaneously
- "What's actually running in prod?" requires querying the cluster, not Git
- Rollback = run old pipeline (or manually edit YAML)

**GitOps:**
```
Developer → opens PR → review → merge to main → ArgoCD syncs → done
```
Benefits:
- Cluster state is always in Git — version controlled, reviewable, auditable
- Every change has a PR review
- Rollback = `git revert <commit>` — instant, clean
- Disaster recovery: if cluster is destroyed, recreate from Git in minutes
- Developers don't need `kubectl` access to production

**The honest tradeoff:** GitOps adds friction to quick emergency changes. If production is on fire and you need to scale immediately, opening a PR is slower than `kubectl scale`. Most teams solve this with an emergency "break glass" procedure — direct kubectl access with required incident ticket, then retroactively commit the change to Git.

---

## Q3: What is Helm and when should you NOT use it?

**Senior Answer:**

Helm is a package manager for Kubernetes. It lets you templatize K8s manifests with Go templating, package them into a "chart," and deploy them with different values per environment.

**What Helm solves:**
- DRY YAML — one template, many deployments
- Release management — `helm install`, `helm upgrade`, `helm rollback`
- Dependency management — your app chart can depend on a PostgreSQL chart
- Community charts — `helm install prometheus prometheus-community/kube-prometheus-stack` gets you a production-grade monitoring stack in one command

**When I'd use Helm:**
- Packaging an app used by multiple teams or environments
- Installing community software (Prometheus, Cert-manager, ArgoCD, NGINX Ingress)
- When you need parameterized deployments with values overrides

**When I'd NOT use Helm:**
- Simple single-environment apps — plain YAML is easier to read and debug than templated YAML
- When you're using Kustomize — Kustomize is simpler for environment-specific overlays without learning Go templating
- When the chart becomes a 500-line YAML spaghetti that nobody can maintain — at that point, split into smaller charts or use a different approach

**The gotcha interviewers love:** Helm `upgrade` doesn't always detect all changes correctly — especially for StatefulSets, CRDs, or resources that Helm doesn't manage directly. Always do a `helm diff` (with the helm-diff plugin) before upgrading in production.

---

## Q4: What is the difference between Helm and Kustomize?

**Senior Answer:**

Both solve "I need different K8s config per environment" but with different philosophies:

**Helm** uses templating — YAML files with `{{ .Values.replicaCount }}` placeholders. You provide a `values.yaml` with the actual values. Powerful but you're essentially writing a mini programming language inside YAML.

**Kustomize** uses overlays — you start with a base YAML (no templates, valid K8s YAML) and patch it for each environment. No special syntax in the base files.

```
Kustomize structure:
base/
  deployment.yaml        ← valid K8s YAML, works as-is
  service.yaml
overlays/
  production/
    kustomization.yaml   ← "use base, but change replicas to 5"
    replica-patch.yaml
  staging/
    kustomization.yaml   ← "use base, but use image tag: staging"
```

**When to use which:**
- **Helm** — when you're packaging something for multiple users/teams who need extensive customization, or when using community charts
- **Kustomize** — when you control all the deployments yourself and just need per-environment differences. Built into `kubectl` natively (`kubectl apply -k .`)

**In practice:** Many teams use both. Kustomize to manage environment overlays at the top level, Helm for third-party dependencies. ArgoCD supports both natively.

---

## Q5: How would you design a zero-downtime deployment pipeline?

**Senior Answer:**

Zero-downtime requires getting several things right simultaneously:

**1. Rolling update strategy in the Deployment:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1         # Spin up 1 new pod before killing old ones
    maxUnavailable: 0   # Never remove old pods before new ones are ready
```
`maxUnavailable: 0` is the key — ensures capacity is maintained throughout.

**2. Readiness probes must be correct:**
New pods must pass readiness before receiving traffic. If your app takes 30 seconds to start, your readiness probe must reflect that. Without this, K8s sends traffic to pods that aren't ready → 500 errors during rollout.

**3. Graceful shutdown:**
When K8s terminates old pods, it sends SIGTERM. Your app must handle SIGTERM — finish processing current requests, then exit. Set `terminationGracePeriodSeconds` longer than your slowest request timeout.
```yaml
spec:
  terminationGracePeriodSeconds: 60  # 60s to finish in-flight requests
```

**4. Pre-stop hook (for Kubernetes load balancer sync):**
There's a race condition: kube-proxy takes time to remove the old pod from the load balancer routing table. A brief pre-stop sleep prevents requests from hitting a pod that's already starting shutdown:
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]  # Wait 5s before starting shutdown
```

**5. Database migrations before code deploy:**
If new code expects a new DB schema, run migrations as a Kubernetes `Job` or `initContainer` before the Deployment rollout. Never ship code and schema changes simultaneously without backward compatibility.

**6. Feature flags:**
Decouple deploy from release. Deploy the new code with the feature disabled. Enable the feature flag. If something goes wrong, disable the flag — no rollback needed.

---

## Q6: A junior dev asks: "Should I put the DB password in the Helm values.yaml file?" How do you respond?

**Senior Answer:**

No, and here's why this matters practically, not just theoretically:

`values.yaml` in a Helm chart almost certainly lives in a Git repository. Git repositories get cloned, forked, leaked, and backed up in ways you can't control. A password in `values.yaml` is a password in Git — permanently, including in the history even if you delete it later.

**What to do instead — three options by maturity:**

**Option 1 — Kubernetes Secrets (basic):**
Store the secret in K8s directly:
```bash
kubectl create secret generic db-creds \
  --from-literal=password=MyPassword123 -n production
```
In `values.yaml`, reference the secret name, not the value:
```yaml
database:
  existingSecret: "db-creds"
  existingSecretKey: "password"
```
The chart uses `secretKeyRef` to inject it. Password never in Git.

**Option 2 — External Secrets Operator (intermediate):**
Store the actual secret in AWS Secrets Manager / Vault. The External Secrets Operator syncs it into K8s Secrets. You commit only the ExternalSecret resource (which contains no sensitive data) to Git.

**Option 3 — Vault Agent Injector (advanced):**
HashiCorp Vault with the agent injector. Secrets are injected directly into the pod's filesystem at startup. Never in etcd at all.

**What I'd implement today:** Option 2 for a team already on AWS/GCP. Option 1 if you're just getting started. Either way — the word "password" should never appear in any file committed to Git.
