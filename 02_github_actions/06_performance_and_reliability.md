# GitHub Actions Performance and Reliability

> **Who this is for**: Teams whose workflows are slow, expensive, flaky, or vulnerable to overlapping changes. Read [Cross-Repository Workflow Orchestration](05_cross_repository_orchestration.md) first.

---

## 1. Optimize the Critical Path

Pipeline duration is the longest dependency path, not the sum of every job:

```text
lint (2m) ───────────────┐
unit (7m) ───────────────┼──> build (8m) ──> smoke (3m)
policy (3m) ─────────────┘

critical path = unit 7m + build 8m + smoke 3m = 18m
```

Measure:

- queue time before a runner starts;
- setup and dependency-install time;
- execution time by command;
- critical-path duration;
- cancellation and rerun rate;
- cache hit rate;
- first-attempt failure and flake rate;
- billable minutes by repository and workflow.

Optimize measured bottlenecks. A cache that saves ten seconds but creates poisoning risk is a poor trade.

---

## 2. Use Concurrency with Correct Semantics

Cancel superseded PR validation:

```yaml
concurrency:
  group: ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

Serialize a deployment:

```yaml
jobs:
  deploy:
    concurrency:
      group: deploy-production-orders
      cancel-in-progress: false
```

GitHub concurrency permits at most one running and one pending job or run per group. A newer pending item replaces an existing pending item. Therefore, `cancel-in-progress: false` protects the running deployment but is not a durable FIFO queue for every release.

```text
release A = running
release B = pending
release C arrives
    → A continues
    → B may be cancelled
    → C becomes pending
```

If every intermediate release must deploy, use an external queue or deployment controller. Most continuous-delivery systems intentionally deploy the newest eligible desired state.

Build concurrency keys from bounded, trusted identifiers. Avoid a user-controlled value that can force unrelated deployments into the same group.

---

## 3. Shape Matrices Deliberately

```yaml
strategy:
  fail-fast: false
  max-parallel: 4
  matrix:
    python: ["3.11", "3.12", "3.13"]
    database: ["postgres-16", "postgres-17"]
    exclude:
      - python: "3.11"
        database: postgres-17
```

| Control | Use |
|---------|-----|
| `fail-fast: true` | Stop sibling combinations after a decisive failure |
| `fail-fast: false` | Gather compatibility evidence across every combination |
| `max-parallel` | Protect rate-limited services and runner capacity |
| `include` / `exclude` | Express supported combinations instead of a wasteful Cartesian product |

Run the primary supported combination on every PR. Move the full compatibility matrix to merge queues, main, or a schedule if it is too expensive for fast feedback.

---

## 4. Cache Dependencies, Not Trust

```yaml
- uses: actions/setup-python@a26af69be951a213d495a4c3e4e4022e16d87065 # v5
  with:
    python-version: "3.12"
    cache: pip
    cache-dependency-path: |
      requirements.txt
      requirements-dev.txt
```

Cache-key inputs should include:

- operating system and architecture;
- runtime/toolchain version;
- dependency lockfile hash;
- build mode when outputs differ;
- a manual schema version when cache layout changes.

Rules:

- a miss must still produce a correct build;
- never store credentials or signed release outputs;
- validate restored executables before privileged use;
- remember that branch cache scopes can expose default-branch caches to pull requests;
- rotate a cache namespace after suspected poisoning.

Use registry-backed BuildKit caches for container layers, but publish the final image separately and identify it by digest.

---

## 5. Use Change Detection without Losing Coverage

For a monorepo:

```text
detect changes
├── compute affected components
├── run their checks in a bounded matrix
├── include dependents of changed shared code
└── report one stable required gate
```

Add a scheduled full build to validate the dependency graph. Track cases where full builds fail but affected-only PR builds passed; they reveal gaps in change detection.

Path filters on an entire required workflow can leave an expected check pending. Prefer an always-created detection job and conditional downstream jobs.

---

## 6. Bound Every Wait

Job-level timeout:

```yaml
jobs:
  integration:
    timeout-minutes: 30
```

Command-level timeouts:

```bash
curl \
  --connect-timeout 5 \
  --max-time 20 \
  --retry 4 \
  --retry-all-errors \
  --retry-delay 2 \
  --fail \
  --silent \
  --show-error \
  https://staging.example.com/health/ready
```

Retry only operations that are idempotent or have an idempotency key. Use exponential backoff with jitter for APIs. Do not retry deterministic compilation or assertion failures.

```text
safe retry:
    read request, idempotent PUT, digest-based deployment

unsafe without protection:
    charge card, append message, create unkeyed resource
```

---

## 7. Retain the Right Evidence

| Artifact | Suggested policy |
|----------|------------------|
| Passing unit-test reports | Short retention |
| Failure logs and screenshots | Long enough for normal investigation |
| Release manifest and provenance | Match product support and audit period |
| Large debug bundles | Upload on failure; scrub secrets |
| Production binary/image | Durable registry retention, not only workflow storage |

```yaml
- uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4
  if: ${{ failure() }}
  with:
    name: integration-debug-${{ github.run_id }}-${{ github.run_attempt }}
    path: test-results/debug/
    if-no-files-found: warn
    retention-days: 14
```

Deleting a workflow run deletes its associated workflow artifacts. Preserve release assets in a registry or release store with explicit lifecycle policy.

---

## 8. Make Cleanup Reliable

Preview environments and temporary accounts need two cleanup paths:

1. event-driven cleanup when a pull request closes;
2. scheduled reconciliation that deletes expired resources missed by events.

Every resource should carry:

```text
repository = acme/orders
run_id     = 8912345678
pr_number  = 482
owner      = application-team
expires_at = 2026-07-31T12:00:00Z
```

Cleanup scripts must reject an empty identifier, verify ownership labels, and limit deletion to the expected scope.

---

## 9. Control Runner and External-Service Capacity

- Use `max-parallel` for large matrices.
- Use runner groups to constrain privileged workloads.
- Separate low-trust validation from deployment runners.
- Monitor job queue time and autoscaler convergence.
- Rate-limit calls to scanners, registries, and cloud APIs.
- Prefer ephemeral runners when self-hosting.
- Prebuild runner images for stable heavy toolchains, but patch them regularly.

Warm persistent runners may reduce setup time but carry workspace and credential-contamination risk. Performance cannot be evaluated separately from the trust model.

---

## 10. Common Failure Modes

**Caching masks undeclared dependencies**

A clean build fails. Run periodic cache-disabled builds and keep reconstruction complete.

**All matrices run on every documentation change**

Compute affected scope and keep one scheduled full validation.

**Retries turn incidents into long queues**

Bound attempts, classify errors, and stop retrying non-transient failures.

**Concurrency is assumed to be a release queue**

Pending work can be replaced. Decide whether the desired state is “latest wins” or whether every change needs durable processing.

**Optimization removes evidence**

Track escaped defects and full-build mismatches. Faster feedback is useful only while risk remains controlled.

---

## 11. References

- [Control workflow and job concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)
- [Dependency caching](https://docs.github.com/en/actions/concepts/workflows-and-actions/dependency-caching)
- [Workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts)
- [Self-hosted runners reference](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)

---

**Next**: [Permissions, Secrets, and OIDC](../03_security_and_supply_chain/01_permissions_secrets_and_oidc.md)
