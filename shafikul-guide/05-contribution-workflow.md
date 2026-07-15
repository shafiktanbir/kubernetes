# 05 — Contribution Workflow
> The exact steps from "I want to contribute" to "my PR is merged."
> Read once end-to-end, then follow step-by-step on your first PR.

---

## Phase 0: Sign the CLA — Do This Before Anything Else

**The CNCF Contributor License Agreement (CLA) is required. Without it, the PR bot will block every PR you open, permanently, until you sign.**

1. Go to: https://identity.linuxfoundation.org/projects/cncf
2. Sign in with the **same GitHub account** you'll use to open PRs
3. Sign the Individual CLA (takes 5 minutes)
4. Done — it applies to all CNCF projects (Kubernetes, Prometheus, Envoy, etc.) permanently

> If you're contributing on behalf of a company, your employer must sign the Corporate CLA separately. Ask your manager.

---

## Phase 1: Find the Right Issue

### Where to Look

```bash
# Good first issues (labeled by maintainers)
https://github.com/kubernetes/kubernetes/issues?q=label%3A%22good+first+issue%22+is%3Aopen

# Help wanted (slightly harder, still newcomer-friendly)
https://github.com/kubernetes/kubernetes/issues?q=label%3A%22help+wanted%22+is%3Aopen

# Filter by area (example: scheduler)
https://github.com/kubernetes/kubernetes/issues?q=label%3Aarea%2Fscheduler+label%3A%22good+first+issue%22+is%3Aopen
```

### What Makes a Good First Issue

| Type | Why it's good |
|------|--------------|
| Logging improvements | Small, testable, clear scope |
| Error message improvements | Same |
| Documentation/comment fixes | No test requirement |
| Typo fixes | Get familiar with the PR process |
| Unit test additions | Learn the package deeply |
| Small bug fixes (with existing test) | Shows real impact |

**Avoid for your first PR:**
- New API fields (needs KEP + API review)
- Performance changes (needs benchmarks)
- Architecture refactors (needs design doc)
- Multi-package changes (hard to review)

### Before Claiming an Issue

1. Read the issue fully including all comments — it may already be solved or assigned
2. Check if there's a linked PR — someone may already be working on it
3. Post: *"I'd like to work on this. I'm looking at [specific file/function]. Is this still unassigned?"*
4. Wait for a maintainer to confirm — don't submit a PR to a claimed issue

---

## Phase 2: Set Up Your Environment

### Fork and Clone

```bash
# 1. Fork on GitHub: https://github.com/kubernetes/kubernetes → Fork

# 2. Clone YOUR fork (not the upstream)
git clone https://github.com/<your-username>/kubernetes.git
cd kubernetes

# 3. Add upstream remote
git remote add upstream https://github.com/kubernetes/kubernetes.git

# 4. Verify remotes
git remote -v
# origin    https://github.com/<your-username>/kubernetes.git
# upstream  https://github.com/kubernetes/kubernetes.git
```

### Install Prerequisites

```bash
# Go (check required version)
cat .go-version    # e.g., 1.22.4

# etcd (for local cluster and integration tests)
hack/install-etcd.sh
export PATH=$PATH:$(pwd)/third_party/etcd

# Build tools
sudo apt-get install -y build-essential jq
```

### Verify Your Setup

```bash
# Build everything (takes 5-10 minutes first time)
make

# Run a quick unit test to verify Go is working
make test WHAT=./pkg/scheduler/... GOFLAGS=-v 2>&1 | tail -20
```

---

## Phase 3: Make Your Change

### Create a Branch

```bash
# Always branch from a fresh main
git fetch upstream
git checkout -b fix/scheduler-improve-error-message upstream/main
```

Branch naming convention:
```
fix/<short-description>     ← bug fixes
feature/<short-description> ← new features
docs/<short-description>    ← documentation
test/<short-description>    ← test additions
```

### Read the Code Before Writing

Before changing anything:
1. Find the function/file responsible (use IDE "Go to Definition")
2. Read 50 lines above and below your change point
3. Find existing tests for that function
4. Run existing tests to confirm they pass:
   ```bash
   make test WHAT=./pkg/<package>/...
   ```

### License Header (Required on Every New .go File)

If you create a new `.go` file, it must start with:

```go
/*
Copyright 2025 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/
```

Template is at: `hack/boilerplate/boilerplate.go.txt`

### Logging Style

```go
import "k8s.io/klog/v2"

// Structured logging — always use InfoS/ErrorS not Info/Error
klog.V(4).InfoS("Scheduling pod", "pod", klog.KObj(pod), "node", nodeName)
klog.ErrorS(err, "Failed to sync pod", "pod", klog.KObj(pod))

// klog.KObj creates a {namespace/name} representation
// klog.KRef for references you don't have the object for
klog.V(2).InfoS("Selected node", "pod", klog.KObj(pod), "node", klog.KRef("", nodeName))
```

Verbosity levels:
- `V(0)` / default — always shown, important events
- `V(2)` — notable but not unusual
- `V(4)` — debug info
- `V(6)` — very verbose trace info

### Error Messages

```go
// Good: specific and actionable
return fmt.Errorf("cannot schedule pod %s/%s: node %s has insufficient memory (requested: %v, available: %v)",
    pod.Namespace, pod.Name, nodeName, requested, available)

// Bad: vague
return fmt.Errorf("scheduling failed")
```

### Do NOT

```go
// Never edit generated files — they start with this comment:
// Code generated by ... DO NOT EDIT.

// Never use panic() unless the state is truly unrecoverable
// (initialization failures are OK, runtime errors are not)

// Never log sensitive data (tokens, passwords, secrets content)

// Never add global mutable state
```

---

## Phase 4: Write or Update Tests

### Every change needs a test. No exceptions.

**Unit test location:** Same package, file ending in `_test.go`
```
pkg/scheduler/schedule_one.go       ← your change
pkg/scheduler/schedule_one_test.go  ← your test (already exists, add to it)
```

**Test pattern used in Kubernetes:**

```go
func TestMyFunction(t *testing.T) {
    tests := []struct {
        name     string
        input    SomeType
        expected SomeResult
        wantErr  bool
    }{
        {
            name:     "happy path",
            input:    SomeType{...},
            expected: SomeResult{...},
        },
        {
            name:    "error case",
            input:   SomeType{invalid: true},
            wantErr: true,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            result, err := MyFunction(tc.input)
            if (err != nil) != tc.wantErr {
                t.Errorf("MyFunction() error = %v, wantErr = %v", err, tc.wantErr)
                return
            }
            if result != tc.expected {
                t.Errorf("MyFunction() = %v, want %v", result, tc.expected)
            }
        })
    }
}
```

**Run your tests:**
```bash
# Run all tests in a package
make test WHAT=./pkg/scheduler/...

# Run a specific test
make test WHAT=./pkg/scheduler/... GOFLAGS="-v -run TestScheduleOne"
```

### Integration Tests (if your change affects API server behavior)

```bash
make test-integration WHAT=./test/integration/scheduler/...
```

---

## Phase 5: Verify Your Changes

Run these **before every push**:

```bash
# 1. Format code
hack/verify-gofmt.sh
# OR auto-fix:
gofmt -w ./pkg/scheduler/

# 2. Run the linter
hack/verify-golangci-lint.sh

# 3. Check imports are sorted correctly
hack/verify-imports.sh

# 4. If you changed generated files (you shouldn't have):
make update

# 5. Run all verifiers at once (what CI runs):
make verify

# 6. Run unit tests for your package:
make test WHAT=./pkg/<your-package>/...
```

If `make verify` passes locally, CI will very likely pass too.

---

## Phase 6: Commit Your Change

### Commit Message Format

```
component: short description in imperative mood, ≤72 chars

Longer explanation of WHY this change was made.
What problem does it solve?
What approach was taken and why?

Fixes #<issue-number>   ← only if there IS an issue to fix
```

**Note: Do NOT add `Fixes #123` if there is no issue. Do not add `@mentions`. Do not add `Co-authored-by:`.**

**Good examples:**
```
scheduler: include container names in NodeResourcesFit failure message

When a pod fails to schedule due to resource constraints, the error
message previously showed only the total resource shortage. This made
it difficult to identify which container was causing the issue.

This change adds container-level resource breakdown to the failure
reason, showing the required amounts per container.

Fixes #98765
```

```
kubelet: fix nil pointer in eviction manager when stats are missing

If cAdvisor fails to report memory stats for a container, the eviction
manager would panic trying to dereference a nil stats pointer.

Add a nil check before accessing stats and log a warning instead.
```

```
kubectl: update help text for --output flag to list all valid values

The --output flag accepted several undocumented values. Added all valid
values (json, yaml, wide, name, custom-columns, jsonpath) to the help text.
```

### Commit:

```bash
git add -p    # review each change before staging (don't git add .)
git commit    # opens editor for commit message
```

**Why `git add -p` instead of `git add .`:**
- Forces you to review each diff
- Prevents accidentally committing debug code or temp files
- Keeps commits focused

---

## Phase 7: Open the Pull Request

### Push Your Branch

```bash
git push origin fix/scheduler-improve-error-message
```

### Create the PR

Go to your fork on GitHub → "Compare & pull request"

**PR Title:** Same as your commit message first line
```
scheduler: include container names in NodeResourcesFit failure message
```

**PR Description template:**
```markdown
## What this PR does / why we need it

[Explain the problem and your solution in 2-4 sentences]

## Which issue(s) this PR fixes

Fixes #98765

## Special notes for your reviewer

[Anything the reviewer needs to know: alternative approaches considered,
known limitations, follow-up work needed]

## Does this PR introduce a user-facing change?

[Yes/No — if yes, describe it for the CHANGELOG]
```

### After Opening the PR

1. CI will start running automatically. Check the status after ~15 minutes.
2. If CI fails, read the log and fix it. Common failures:
   - `verify-gofmt` — run `gofmt -w` on your files
   - `verify-imports` — run `goimports -w` on your files
   - `unit tests` — a test you broke
3. Post in the relevant SIG Slack channel:
   ```
   Hey #sig-scheduling — opened a PR to improve error messages in NodeResourcesFit.
   Would appreciate a review when someone has time: <link>
   ```
4. **Be patient.** Maintainers are volunteers. Response time: 1 day to 2 weeks.

---

## Phase 8: Respond to Review

### When a Reviewer Comments

- **Respond to every comment** — even if just "done" or "agree"
- For disagreements: explain your reasoning, don't just revert
- Push fixes as **additional commits** (do not force-push, it makes reviews harder)
- After addressing feedback, post a summary:
  ```
  @reviewer — addressed all comments:
  - Fixed the nil check as suggested
  - Kept the logging at V(4) because this is a debug path
  - Added test case for the empty stats scenario
  ```

### If Review Stalls

After 2 weeks with no response:
1. Ping the PR with a brief comment: "Gentle ping — is there anything I should fix?"
2. Or post in the SIG Slack channel: "PR #12345 has been open for 2 weeks, any chance of a review?"

### Rebase When Needed

If your branch gets out of date with main:
```bash
git fetch upstream
git rebase upstream/main
# resolve any conflicts
git push origin fix/my-branch --force-with-lease
```

Use `--force-with-lease`, never `--force`. It's safer — it fails if someone else pushed to your branch.

---

## Phase 9: Merge

When your PR has:
- `/lgtm` from a reviewer
- `/approve` from an approver
- All CI checks passing
- No `/hold` label

Prow (the bot) merges it automatically. You'll see:
```
Tide merging your PR.
```

**Congratulations — you're a Kubernetes contributor.**

---

## Daily Workflow Summary

```bash
# Morning: sync with upstream
git fetch upstream
git rebase upstream/main

# Work on your change
# ... edit files ...

# Test continuously
make test WHAT=./pkg/<package>/... GOFLAGS=-v

# Before pushing
make verify

# Push
git add -p
git commit
git push origin <branch>
```

---

## Common Mistakes and How to Avoid Them

| Mistake | Prevention |
|---------|-----------|
| Not signing the CLA before opening a PR | Sign at https://identity.linuxfoundation.org/projects/cncf first |
| Editing `zz_generated.*` | Never. Run `make update` instead |
| Running `go mod tidy` | Use `hack/pin-dependency.sh` + `hack/update-vendor.sh` |
| Committing without running tests | `make test WHAT=./pkg/<package>/...` before every commit |
| Force-pushing to origin | Never during review. Use `--force-with-lease` only if needed |
| Missing license header on new files | Copy from `hack/boilerplate/boilerplate.go.txt` |
| Huge PRs (>500 line diffs) | Split into multiple focused PRs |
| No issue linked for non-trivial change | Always link or create an issue first |
| Ignoring CI failures | Fix them. Don't merge with red CI. |
| Changing unrelated files | Revert. Keep PRs focused. |

---

## Cheat Sheet

```bash
# Build
make

# Test one package
make test WHAT=./pkg/scheduler/...

# Test one specific test function
make test WHAT=./pkg/scheduler/... GOFLAGS="-v -run TestScheduleOne"

# Integration tests
make test-integration WHAT=./test/integration/scheduler/...

# All verifiers
make verify

# Regenerate all generated code
make update

# Start local cluster
hack/local-up-cluster.sh

# Install etcd locally
hack/install-etcd.sh && export PATH=$PATH:$(pwd)/third_party/etcd

# Check Go version required
cat .go-version

# Find which SIG owns a file
cat pkg/scheduler/OWNERS

# Search for a function across the codebase
grep -r "func SyncPod" --include="*.go" .
```
