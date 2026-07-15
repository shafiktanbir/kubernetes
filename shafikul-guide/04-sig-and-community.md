# 04 — SIG Groups and Community
> How Kubernetes is governed, which groups own what, how to talk to them,
> and how to show up without looking like a newcomer who hasn't done homework.

---

## What Is a SIG?

SIG = **Special Interest Group**. Kubernetes development is organized into SIGs. Each SIG:
- Owns a part of the codebase
- Runs regular public meetings (video call, recorded)
- Has a mailing list
- Has a Slack channel
- Reviews and approves PRs in their area
- Writes design docs (KEPs) for new features

There are also **Working Groups (WG)** — temporary groups tackling a cross-SIG problem, dissolved when done.

---

## SIG Map — Which SIG Owns What

| SIG | Code areas | Slack | Meetings |
|-----|-----------|-------|---------|
| **SIG Node** | kubelet, CRI, node lifecycle, resource management, pod lifecycle | `#sig-node` | Weekly Tue |
| **SIG Scheduling** | kube-scheduler, scheduler plugins, pod placement | `#sig-scheduling` | Weekly Wed |
| **SIG API Machinery** | kube-apiserver, client-go, apimachinery, CRDs, API review | `#sig-api-machinery` | Biweekly Thu |
| **SIG Apps** | Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob | `#sig-apps` | Biweekly Mon |
| **SIG Network** | kube-proxy, Services, DNS (CoreDNS), NetworkPolicy, Ingress | `#sig-network` | Biweekly Thu |
| **SIG Storage** | PV/PVC, StorageClass, CSI, volume plugins | `#sig-storage` | Biweekly Thu |
| **SIG Auth** | RBAC, authentication, admission, service accounts, secrets | `#sig-auth` | Biweekly Wed |
| **SIG CLI** | kubectl, kubeadm | `#sig-cli` | Biweekly Wed |
| **SIG Cluster Lifecycle** | kubeadm, cluster setup/teardown, upgrade | `#sig-cluster-lifecycle` | Weekly Tue |
| **SIG Instrumentation** | Metrics, logging, tracing, Events API | `#sig-instrumentation` | Biweekly Mon |
| **SIG Testing** | e2e framework, test infrastructure, flaky tests | `#sig-testing` | Biweekly Tue |
| **SIG Scalability** | Performance, large-cluster behavior, benchmarks | `#sig-scalability` | Biweekly Fri |
| **SIG Architecture** | Overall design, API conventions, KEP process | `#sig-architecture` | Biweekly Thu |
| **SIG Release** | Release process, branching, changelog | `#sig-release` | Weekly Mon |
| **SIG Security** | Security issues, CVEs, vulnerability response | `#sig-security` | Biweekly |

**Full list:** https://github.com/kubernetes/community/blob/master/sig-list.md

---

## How to Engage a SIG — Step by Step

### Before Your First Contact

1. **Read the SIG's README** — every SIG has a directory in `kubernetes/community`:
   ```
   https://github.com/kubernetes/community/tree/master/sig-<name>
   ```
   Read it. It lists what they own, their charter, their meetings, their leads.

2. **Lurk on Slack first** — join the channel, read recent messages, understand the tone before posting.

3. **Read recent meeting notes** — meeting notes are public. Find them in the SIG's Google Doc or GitHub. You'll know what they're currently working on.

4. **Know the terminology** — use the vocabulary from `01-k8s-big-picture.md`. Saying "informer" instead of "watcher object" signals you've done homework.

### Your First Message in Slack

Bad:
```
Hi, I'm new and want to contribute to Kubernetes. Where should I start?
```

Good:
```
Hi SIG Node folks — I'm looking to make my first contribution.
I've been reading through pkg/kubelet/eviction and noticed the error message
at L412 of eviction_manager.go says "failed to observe eviction thresholds"
but doesn't include the threshold values. Is this a known issue or would a
patch to improve that log message be welcome?
```

The difference: you show you have already read specific code and have a concrete proposal.

### Your First Mailing List Post

Join via Google Groups: `kubernetes-sig-<name>@googlegroups.com`

Introduce yourself with:
- What you've been learning (specific files/packages)
- What subsystem you want to contribute to and why
- A concrete question or proposal

Example:
```
Subject: [New Contributor] Looking to contribute to scheduler plugins

Hi,

I'm Shafikul, a software engineer who has been studying the Kubernetes scheduler.
I've read through pkg/scheduler/framework/interface.go and a few plugin implementations
(NodeResourcesFit, NodeAffinity).

I noticed that when a pod fails to schedule due to resource constraints, the event
message only shows the total shortage but not which specific container caused it.
I was thinking of improving the error message in noderesources/fit.go to include
container-level resource requirements.

Is this something SIG Scheduling would be interested in? Or is there an existing
issue tracking this? Happy to write a KEP if needed or just submit a PR if it's
small enough to not need one.

Thanks,
Shafikul
```

### Attending a SIG Meeting

1. Find the calendar: https://calendar.google.com/calendar/u/0/embed?src=sig_scheduling@kubernetes.io (substitute sig name)
2. Meetings are on Zoom, link is in the SIG README
3. **You do not need to speak** — listening is valuable
4. Meeting notes are taken publicly in a shared Google Doc
5. If you have a question/update, add it to the agenda doc before the meeting

---

## How KEPs Work

KEP = **Kubernetes Enhancement Proposal**. Required for any significant feature.

**Repository:** https://github.com/kubernetes/enhancements

**KEP structure:**
```
keps/
└── sig-scheduling/
    └── 1234-my-feature/
        ├── README.md          ← The proposal
        ├── kep.yaml           ← Metadata (status, authors, reviewers)
        └── test-plan.md       ← How the feature will be tested
```

**KEP lifecycle:**
```
provisional → implementable → implemented → deferred / withdrawn
```

**For contributors:** You don't need to write a KEP for bug fixes, doc improvements, or small enhancements. You need one for:
- New API fields or types
- New features that change behavior
- Changes to existing API semantics

When unsure: ask in Slack or the mailing list.

---

## How PRs Are Approved

Kubernetes uses **Prow** (the CI/CD bot) and **OWNERS files** for PR workflow.

### OWNERS Files

Every directory has an `OWNERS` file:
```yaml
approvers:
  - alicejones      # can merge PRs
reviewers:
  - bobjohnson      # can /lgtm PRs
```

Your PR needs:
1. `/lgtm` from a reviewer in the modified files' OWNERS
2. `/approve` from an approver in the modified files' OWNERS

### Prow Commands (Post as PR comments)

| Command | Who can use it | Effect |
|---------|---------------|--------|
| `/lgtm` | Reviewer | Adds `lgtm` label |
| `/approve` | Approver | Approves + merges |
| `/hold` | Anyone | Blocks merge |
| `/retest` | Anyone | Retriggers failed CI |
| `/assign @username` | Anyone | Assigns reviewer |
| `/cc @username` | Anyone | Requests review |
| `/kind bug` | Anyone | Labels as bug |
| `/kind feature` | Anyone | Labels as feature |
| `/area scheduler` | Anyone | Labels with area |

### CI Checks

Every PR runs:
- `pull-kubernetes-unit` — unit tests
- `pull-kubernetes-integration` — integration tests
- `pull-kubernetes-e2e` — e2e tests (subset)
- `pull-kubernetes-verify` — all verifiers (formatting, imports, generated code)

A failing CI check blocks merge. Fix it by pushing a new commit.

---

## Useful Links

### Communication
- **Slack:** https://kubernetes.slack.com (get invite at https://slack.k8s.io)
- **Mailing lists:** https://groups.google.com/a/kubernetes.io
- **Community calendar:** https://kubernetes.io/community
- **Discuss forum:** https://discuss.kubernetes.io

### Code
- **Main repo:** https://github.com/kubernetes/kubernetes
- **Community repo:** https://github.com/kubernetes/community
- **Enhancements (KEPs):** https://github.com/kubernetes/enhancements
- **Test results dashboard:** https://testgrid.k8s.io
- **Code search:** https://cs.k8s.io (powered by OpenGrok)

### Contributor Path
- **Contributor guide:** https://github.com/kubernetes/community/blob/master/contributors/guide/README.md
- **Contributor ladder:** https://github.com/kubernetes/community/blob/master/community-membership.md
- **Good first issues:** https://github.com/kubernetes/kubernetes/issues?q=label%3A%22good+first+issue%22
- **Help wanted:** https://github.com/kubernetes/kubernetes/issues?q=label%3A%22help+wanted%22

---

## The Contributor Ladder

| Level | Requirements | Permissions |
|-------|-------------|------------|
| **Contributor** | 1 merged PR | Can `/retest` and open issues |
| **Member** | 5+ contributions, 2 sponsors, active for 3 months | Can be assigned issues, `/lgtm` their own PRs |
| **Reviewer** | History of quality reviews in an area, sponsored by Approver | `/lgtm` PRs in their area |
| **Approver** | History as Reviewer, deep area knowledge | `/approve` PRs, listed in OWNERS |
| **Subproject owner** | Long-term maintainer | Manage the subproject |

**Realistic timeline for first merge:** 1–4 weeks after finding the right issue.
**Realistic timeline for Membership:** 3–6 months of active contribution.

---

## Which SIG Should You Start With?

Based on your interest:

| If you want to work on… | Start with |
|------------------------|-----------|
| How pods get placed | SIG Scheduling |
| How containers run on nodes | SIG Node |
| How the API works, client libraries | SIG API Machinery |
| kubectl improvements | SIG CLI |
| Deployment/ReplicaSet/Job behavior | SIG Apps |
| Networking, Services | SIG Network |
| Storage, volumes | SIG Storage |
| RBAC, tokens, security | SIG Auth |
| Performance, 5000-node clusters | SIG Scalability |
| Test infrastructure, flaky tests | SIG Testing |

**Recommendation for first-time contributors:** SIG CLI or SIG Scheduling.
- SIG CLI: kubectl is self-contained, familiar, well-tested
- SIG Scheduling: plugin framework is clean and well-documented

---

## Things That Signal You're a Serious Contributor

In Slack/meetings, these signal you've done your homework:
- Reference specific file paths and line numbers
- Mention the relevant SIG correctly
- Know whether something needs a KEP
- Use terms like "informer", "reconcile loop", "admission", "feature gate" correctly
- Ask "is there an existing issue for this?" before proposing something
- Know the difference between `staging/` and `pkg/`

Things to avoid:
- Asking "where do I start?" without context
- Proposing major architecture changes as a first PR
- Opening a PR without a linked issue (for non-trivial changes)
- Ignoring CI failures
