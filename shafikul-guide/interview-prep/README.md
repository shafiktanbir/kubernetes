# 🎯 Kubernetes Interview Prep — Senior Level

> Covers Backend Engineer + DevOps/Platform Engineer interview questions.
> These are the questions that actually get asked at companies like Amazon, Google, Shopify, Cloudflare, startups, and mid-size tech companies.

---

## How to Use This

1. **Do the labs first** (`../k3d-labs/`) — you need hands-on experience to answer these convincingly
2. **Read each answer carefully** — notice HOW the answer is structured
3. **Practice saying it out loud** — interviews are verbal
4. **Add your own lab experience** — "I actually saw this when I did Lab 09 locally..."

---

## Question Categories

| File | Topics | Level |
|------|--------|-------|
| [01-core-concepts.md](./01-core-concepts.md) | Pods, Deployments, Services, Namespaces | ⭐ Must Know |
| [02-networking.md](./02-networking.md) | Ingress, CNI, DNS, Service types | ⭐ Must Know |
| [03-storage.md](./03-storage.md) | PV, PVC, StorageClass, StatefulSets | ⭐ Must Know |
| [04-security.md](./04-security.md) | RBAC, Secrets, NetworkPolicy, PSA | ⭐ Must Know |
| [05-scaling-scheduling.md](./05-scaling-scheduling.md) | HPA, VPA, Affinity, Taints/Tolerations | ⭐⭐ Senior Level |
| [06-troubleshooting.md](./06-troubleshooting.md) | Debugging, CrashLoopBackOff, OOMKill | ⭐⭐ Senior Level |
| [07-cicd-gitops.md](./07-cicd-gitops.md) | ArgoCD, Helm, GitOps, CI/CD patterns | ⭐⭐ Senior Level |
| [08-architecture.md](./08-architecture.md) | Control plane, etcd, design decisions | ⭐⭐⭐ Principal Level |

---

## Senior Engineer Answer Formula

Every strong interview answer follows this pattern:

```
1. Define it clearly (1 sentence)
2. Explain the problem it solves (the WHY)
3. Give a concrete example or use case
4. Mention a tradeoff or gotcha (shows depth)
5. Optional: what you did in practice
```

Interviewers remember candidates who explain WHY, not just WHAT.

---

## Red Flags That Kill Interviews

❌ "I know it but I can't explain it"
❌ Only memorizing definitions, no practical experience
❌ Can't debug a CrashLoopBackOff
❌ Doesn't know difference between ClusterIP and NodePort
❌ Never wrote a YAML from scratch

✅ What makes you stand out:
- "I've seen this fail in production / my lab..."
- Mentioning tradeoffs unprompted
- Asking clarifying questions before answering
- Knowing when NOT to use K8s
