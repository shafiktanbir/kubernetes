# Shafikul's Kubernetes Contributor Guide

Personal learning docs for navigating the Kubernetes codebase and community.
Everything here is tied to the actual repo at the parent directory.

---

## Start Here → Daily Plan

**Open this every day. Follow it. No decisions needed.**

| File | Purpose |
|------|---------|
| ⭐ **[07-daily-plan.md](./07-daily-plan.md)** | **30-day plan. 2 hours/day. Day-by-day tasks with checkboxes.** |

---

## Reference Docs (used during the daily plan)

| # | File | Purpose |
|---|------|---------|
| 1 | [01-k8s-big-picture.md](./01-k8s-big-picture.md) | High-level architecture — what K8s is and how it thinks |
| 2 | [02-codebase-map.md](./02-codebase-map.md) | Repository layout, every directory explained |
| 3 | [03-request-flow.md](./03-request-flow.md) | Deep dive: what happens on `kubectl apply` |
| 4 | [04-sig-and-community.md](./04-sig-and-community.md) | SIG groups, how to join, how to talk to them |
| 5 | [05-contribution-workflow.md](./05-contribution-workflow.md) | Step-by-step: from issue to merged PR |
| 6 | [06-official-resources.md](./06-official-resources.md) | Every official link — docs, GitHub repos, SIG channels |

---

## Quick Reference

```bash
# Run unit tests for a package
make test WHAT=./pkg/scheduler/... GOFLAGS=-v

# Start a local cluster
hack/local-up-cluster.sh

# Run all verifiers before pushing
make verify

# Regenerate all generated code
make update
```

> Do not edit `zz_generated.*`, `generated.pb.go`, `go.mod`, or `go.sum` by hand.
