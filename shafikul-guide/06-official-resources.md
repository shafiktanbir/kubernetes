# 06 — Official Resources & Links
> Every official link you need. Bookmarked and organized by topic.
> Open these alongside the code — don't just read docs in isolation.

---

## Official Documentation

| Resource | URL | What it's for |
|----------|-----|--------------|
| Kubernetes Docs (main) | https://kubernetes.io/docs | Concepts, tasks, API reference |
| API Reference (latest) | https://kubernetes.io/docs/reference/kubernetes-api/ | Every API object, every field |
| Concepts | https://kubernetes.io/docs/concepts/ | Deep explanations of how K8s works |
| Tasks | https://kubernetes.io/docs/tasks/ | How-to guides |
| Tutorials | https://kubernetes.io/docs/tutorials/ | Hands-on walkthroughs |
| Glossary | https://kubernetes.io/docs/reference/glossary/ | Official definitions of every term |
| Component reference | https://kubernetes.io/docs/reference/command-line-tools-reference/ | CLI flags for every binary |
| kubectl reference | https://kubernetes.io/docs/reference/kubectl/ | Every kubectl subcommand |
| Feature gates | https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/ | Every feature gate, which version it's in |

---

## Contributor Documentation (Read These First)

| Resource | URL | What it's for |
|----------|-----|--------------|
| **Contributor Guide** | https://github.com/kubernetes/community/blob/master/contributors/guide/README.md | The official start-here guide |
| **Community Membership** | https://github.com/kubernetes/community/blob/master/community-membership.md | Member → Reviewer → Approver ladder |
| **Development Guide** | https://github.com/kubernetes/community/blob/master/contributors/devel/development.md | How to build and test locally |
| **SIG Architecture: API Conventions** | https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md | Rules for designing Kubernetes APIs |
| **SIG Architecture: API Changes** | https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md | How to add/change API fields safely |
| **Coding Conventions** | https://github.com/kubernetes/community/blob/master/contributors/guide/coding-conventions.md | Go style rules specific to K8s |
| **Testing Guide** | https://github.com/kubernetes/community/blob/master/contributors/devel/sig-testing/testing.md | How to write unit, integration, e2e tests |
| **Pull Request Process** | https://github.com/kubernetes/community/blob/master/contributors/guide/pull-requests.md | From opening to merging |
| **Owners files** | https://github.com/kubernetes/community/blob/master/contributors/guide/owners.md | How OWNERS and approval work |
| **Prow commands** | https://prow.k8s.io/command-help | All `/lgtm`, `/approve`, etc. commands |

---

## GitHub Repositories You'll Use

| Repo | URL | Purpose |
|------|-----|---------|
| **kubernetes/kubernetes** | https://github.com/kubernetes/kubernetes | The main monorepo |
| **kubernetes/community** | https://github.com/kubernetes/community | SIG charters, contributor docs, meeting notes |
| **kubernetes/enhancements** | https://github.com/kubernetes/enhancements | KEPs — design proposals for new features |
| **kubernetes/test-infra** | https://github.com/kubernetes/test-infra | CI infrastructure (Prow, Tide, TestGrid) |
| **kubernetes/website** | https://github.com/kubernetes/website | kubernetes.io documentation source |
| **kubernetes/api** | https://github.com/kubernetes/api | Published copy of staging/src/k8s.io/api |
| **kubernetes/apimachinery** | https://github.com/kubernetes/apimachinery | Published copy of staging/src/k8s.io/apimachinery |
| **kubernetes/client-go** | https://github.com/kubernetes/client-go | Published copy of staging/src/k8s.io/client-go |
| **kubernetes/apiserver** | https://github.com/kubernetes/apiserver | Published copy of staging/src/k8s.io/apiserver |
| **kubernetes/kubectl** | https://github.com/kubernetes/kubectl | Published copy of staging/src/k8s.io/kubectl |
| **kubernetes/sample-controller** | https://github.com/kubernetes/sample-controller | Canonical controller example using client-go |
| **kubernetes/sample-apiserver** | https://github.com/kubernetes/sample-apiserver | Canonical custom API server example |

---

## Finding Your First Issue

| URL | Label | Description |
|-----|-------|-------------|
| https://github.com/kubernetes/kubernetes/issues?q=label%3A%22good+first+issue%22+is%3Aopen | `good first issue` | Explicitly labeled for newcomers |
| https://github.com/kubernetes/kubernetes/issues?q=label%3A%22help+wanted%22+is%3Aopen | `help wanted` | Slightly harder, but still approachable |
| https://github.com/kubernetes/kubernetes/issues?q=label%3Aarea%2Fscheduler+label%3A%22good+first+issue%22+is%3Aopen | scheduler + good first issue | Scheduler-specific |
| https://github.com/kubernetes/kubernetes/issues?q=label%3Aarea%2Fkubelet+label%3A%22good+first+issue%22+is%3Aopen | kubelet + good first issue | Kubelet-specific |
| https://github.com/kubernetes/kubernetes/issues?q=label%3Aarea%2Fkubectl+label%3A%22good+first+issue%22+is%3Aopen | kubectl + good first issue | kubectl-specific |
| https://github.com/kubernetes/kubernetes/issues?q=label%3Akind%2Fbug+label%3A%22good+first+issue%22+is%3Aopen | bug + good first issue | Bug fixes labeled as newcomer-friendly |

---

## Community Communication

| Channel | URL | Purpose |
|---------|-----|---------|
| **Slack** | https://kubernetes.slack.com | Primary day-to-day communication |
| Slack invite | https://slack.k8s.io | Get your Slack invite here first |
| **Mailing lists** | https://groups.google.com/a/kubernetes.io | Formal announcements and proposals |
| **Discuss forum** | https://discuss.kubernetes.io | Longer discussions, questions |
| **Community calendar** | https://kubernetes.io/community/ | SIG meeting schedules and links |
| **YouTube** | https://www.youtube.com/c/KubernetesCommunity | Recorded SIG meetings |
| **Twitter/X** | https://twitter.com/kubernetesio | Official announcements |

### Key Slack Channels

| Channel | Purpose |
|---------|---------|
| `#kubernetes-contributors` | General contributor discussion |
| `#sig-node` | Kubelet, CRI, node features |
| `#sig-scheduling` | Scheduler, pod placement |
| `#sig-api-machinery` | API server, client-go, CRDs |
| `#sig-apps` | Deployment, StatefulSet, Job |
| `#sig-network` | Services, DNS, NetworkPolicy |
| `#sig-storage` | Volumes, CSI, PV/PVC |
| `#sig-auth` | RBAC, authentication, admission |
| `#sig-cli` | kubectl, kubeadm |
| `#sig-testing` | Test infrastructure, flaky tests |
| `#sig-release` | Release process |
| `#pr-reviews` | Cross-SIG PR review requests |
| `#enhancements` | KEP discussions |

---

## Code Navigation Tools

| Tool | URL | What it's for |
|------|-----|--------------|
| **K8s Code Search** | https://cs.k8s.io | Grep across all k8s repos without cloning |
| **Prow Dashboard** | https://prow.k8s.io | CI job status and PR status |
| **TestGrid** | https://testgrid.k8s.io | Test pass/fail history across all jobs |
| **Tide Dashboard** | https://prow.k8s.io/tide | Which PRs are queued to merge |
| **PR Finder** | https://prow.k8s.io/pr | Your PRs and their status |
| **GitHub Insights** | https://k8s.devstats.cncf.io | Contribution stats and trends |

---

## Design Proposals (KEPs)

Every significant feature has a KEP. Reading KEPs explains *why* code is written the way it is.

| Resource | URL |
|----------|-----|
| KEP repository | https://github.com/kubernetes/enhancements |
| KEP template | https://github.com/kubernetes/enhancements/blob/master/keps/NNNN-kep-template/README.md |
| All KEPs by SIG | https://github.com/kubernetes/enhancements/tree/master/keps |
| How to write a KEP | https://github.com/kubernetes/enhancements/blob/master/keps/sig-architecture/0000-kep-process/README.md |

### KEPs Worth Reading (foundational concepts)

| KEP | Topic | Link |
|-----|-------|------|
| Scheduler Framework | How the plugin framework was designed | https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/624-scheduling-framework |
| Server-Side Apply | Why `kubectl apply` was redesigned | https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/555-server-side-apply |
| Ephemeral Containers | Debug containers | https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/277-ephemeral-containers |
| Pod Disruption Budget | Voluntary disruption control | https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/85-disruption-policy |
| CRI (Container Runtime Interface) | Why Docker was replaced | https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2040-kubelet-cri |

---

## Learning Resources (External, High Quality)

### Free

| Resource | URL | Best for |
|----------|-----|---------|
| **Kubernetes the Hard Way** | https://github.com/kelseyhightower/kubernetes-the-hard-way | Understanding every component by building from scratch |
| **CNCF K8s Fundamentals (LFS258)** | https://training.linuxfoundation.org/training/kubernetes-fundamentals/ | Structured learning (free audit) |
| **K8s official tutorials** | https://kubernetes.io/docs/tutorials/ | Guided hands-on in your browser |
| **client-go examples** | https://github.com/kubernetes/client-go/tree/master/examples | Real code showing how to use client-go |
| **sample-controller walkthrough** | https://github.com/kubernetes/sample-controller | The canonical controller example |
| **Programming Kubernetes (book samples)** | https://github.com/programming-kubernetes | Code from the O'Reilly book |
| **Kubernetes source code walkthrough** | https://github.com/kubernetes/community/tree/master/contributors/devel | Devel guides from the team itself |

### YouTube Playlists

| Channel | Content |
|---------|---------|
| CNCF (official) | https://www.youtube.com/c/cloudnativefdn — KubeCon talks |
| Kubernetes Community | https://www.youtube.com/c/KubernetesCommunity — SIG meeting recordings |
| TechWorld with Nana | YouTube search "Kubernetes TechWorld Nana" — beginner-friendly |
| That DevOps Guy | YouTube search "Kubernetes That DevOps Guy" — practical K8s |

---

## Reference: Key Concepts by Official Docs Page

Use these when you need the official definition while reading code:

| Concept | Official page |
|---------|--------------|
| Pods | https://kubernetes.io/docs/concepts/workloads/pods/ |
| Deployments | https://kubernetes.io/docs/concepts/workloads/controllers/deployment/ |
| Services | https://kubernetes.io/docs/concepts/services-networking/service/ |
| RBAC | https://kubernetes.io/docs/reference/access-authn-authz/rbac/ |
| Admission Controllers | https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/ |
| Scheduler | https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/ |
| Scheduler plugins | https://kubernetes.io/docs/reference/scheduling/config/ |
| Kubelet | https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/ |
| Container Runtime Interface | https://kubernetes.io/docs/concepts/architecture/cri/ |
| Persistent Volumes | https://kubernetes.io/docs/concepts/storage/persistent-volumes/ |
| ConfigMaps | https://kubernetes.io/docs/concepts/configuration/configmap/ |
| Secrets | https://kubernetes.io/docs/concepts/configuration/secret/ |
| Namespaces | https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/ |
| Labels and Selectors | https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/ |
| Finalizers | https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#foreground-cascading-deletion |
| Owner References | https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#owners-and-dependents |
| Feature Gates | https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/ |
| API deprecation policy | https://kubernetes.io/docs/reference/using-api/deprecation-policy/ |
| API versioning | https://kubernetes.io/docs/reference/using-api/ |

---

## SIG-Specific Resources

### SIG Scheduling
- Slack: `#sig-scheduling`
- Mailing list: https://groups.google.com/a/kubernetes.io/g/sig-scheduling
- Meeting notes: https://github.com/kubernetes/community/tree/master/sig-scheduling
- Scheduler docs: https://kubernetes.io/docs/reference/scheduling/
- Scheduler config: https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/

### SIG Node
- Slack: `#sig-node`
- Mailing list: https://groups.google.com/a/kubernetes.io/g/sig-node
- Meeting notes: https://github.com/kubernetes/community/tree/master/sig-node
- Node docs: https://kubernetes.io/docs/concepts/architecture/nodes/
- CRI docs: https://kubernetes.io/docs/concepts/architecture/cri/

### SIG API Machinery
- Slack: `#sig-api-machinery`
- Mailing list: https://groups.google.com/a/kubernetes.io/g/sig-apimachinery
- Meeting notes: https://github.com/kubernetes/community/tree/master/sig-api-machinery
- API conventions: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md

### SIG CLI
- Slack: `#sig-cli`
- Mailing list: https://groups.google.com/a/kubernetes.io/g/sig-cli
- Meeting notes: https://github.com/kubernetes/community/tree/master/sig-cli
- kubectl docs: https://kubernetes.io/docs/reference/kubectl/

### SIG Apps
- Slack: `#sig-apps`
- Mailing list: https://groups.google.com/a/kubernetes.io/g/sig-apps
- Meeting notes: https://github.com/kubernetes/community/tree/master/sig-apps
- Workload docs: https://kubernetes.io/docs/concepts/workloads/

---

## Certification (Optional, Not Required to Contribute)

| Cert | URL | Relevant? |
|------|-----|----------|
| CKA (Certified Kubernetes Administrator) | https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/ | Useful for understanding operational context |
| CKAD (Certified Kubernetes Application Developer) | https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/ | Useful for understanding developer perspective |
| CKS (Certified Kubernetes Security Specialist) | https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/ | Useful if contributing to SIG Auth/Security |

> Note: Certifications are NOT required for contributing. They test user skills, not source code knowledge. Only pursue if you want the credential.

---

## Daily Bookmarks (Minimal Set)

Bookmark exactly these 8 URLs. You will use them every day:

1. https://github.com/kubernetes/kubernetes — main repo
2. https://cs.k8s.io — code search across all k8s repos
3. https://kubernetes.io/docs/concepts/ — official concepts docs
4. https://prow.k8s.io/pr — your PR status
5. https://github.com/kubernetes/kubernetes/issues?q=label%3A%22good+first+issue%22+is%3Aopen — first issues
6. https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md — API conventions (read weekly)
7. https://github.com/kubernetes/enhancements — KEPs
8. https://kubernetes.slack.com — Slack
