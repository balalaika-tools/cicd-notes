# GitHub Actions Performance and Reliability

> **Who this is for**: Teams whose workflows are slow, expensive, flaky, or vulnerable to overlapping changes. Read [Cross-Repository Workflow Orchestration](05_cross_repository_orchestration.md) first.

## The short version

A workflow with five 2–8 minute jobs doesn't finish in 2–8 minutes, and it doesn't finish in the sum of all five either — it finishes in whichever *chain* of dependent jobs takes longest, the **critical path**. Speeding up a job off that chain does nothing to total duration; speeding up a job on it shortens the whole run by the same amount.

**What you need (3 things):**

1. Per-job duration for every job in the workflow — queue time, setup, and execution.
2. The dependency graph between jobs (`needs:`), not just the flat job list.
3. At least one real run to measure, so the predicted critical path is checked against it.

**Worked example:**

```text
lint (2m) ───────────────┐
unit (7m) ───────────────┼──> build (8m) ──> smoke (3m)
policy (3m) ─────────────┘

lint, unit, and policy start together; build needs all three to finish first.
unit is the slowest of the three (7m), so build waits on unit — not on lint or policy.
smoke starts only once build succeeds.

critical path = unit 7m + build 8m + smoke 3m = 18m
naive sum of all five jobs = 2m + 7m + 3m + 8m + 3m = 23m — 5m longer than the critical path
```

**Success signal:** total workflow duration converges on 18m, and shaving time off `lint` or `policy` (2m, 3m) leaves that 18m unchanged — only `unit`, `build`, or `smoke` can shorten it.

**Not handled yet:** correct concurrency semantics ([§2](#2-cancel-in-progress-serializes-it-doesnt-queue-every-release)), matrix cost control ([§3](#3-an-unshaped-matrix-multiplies-cost-faster-than-coverage)), caching without trusting poisoned input ([§4](#4-a-correct-cache-key-still-isnt-a-trust-boundary)), and reliably verified cleanup of temporary resources ([§8](#8-cleanup-is-reliable-only-if-you-verify-the-delete)).

---

## 1. The Critical Path Decides Duration, Not the Job List

> **Core:** the worked example in the short version above is the whole idea — only the jobs on the longest dependency chain affect total duration. Everything below is about finding that chain reliably and only spending optimization effort on it.

Before optimizing anything, measure:

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

## 2. cancel-in-progress Serializes; It Doesn't Queue Every Release

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

By default, a concurrency group holds at most one running job or run and one pending one — this is `queue: single` behavior, and it applies even when you never write `queue` at all. A newer pending item replaces an existing pending one, so `cancel-in-progress: false` protects the *running* deployment but does not guarantee every release in between actually ships:

```text
release A = running
release B = pending
release C arrives
    → A continues
    → B is replaced (never runs)
    → C becomes pending
```

If every intermediate release must deploy, set `queue: max` on the group instead of standing up an external queue or deployment controller:

```yaml
concurrency:
  group: deploy-production-orders
  cancel-in-progress: false
  queue: max
```

`queue: max` holds up to 100 pending jobs or runs per group instead of one, so releases B and C above would both wait their turn instead of B being discarded ([GitHub Docs: Control workflow and job concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency), checked 2026-08-14). Ordering inside that queue is **first-in, first-out (FIFO)** — the item that started *waiting* first runs first, not the item whose workflow was *dispatched* first; a run still finishing its setup steps when a second run starts waiting can leave that second run completing before it, purely because it entered the wait state earlier.

> **Production:** `queue: max` and `cancel-in-progress: true` are incompatible — you can't tell GitHub to both hold every pending run and cancel superseded ones on the same group, and the configuration is rejected. Pick one policy per group: `cancel-in-progress: true` for validation runs you want replaced, `queue: max` for releases you want preserved.

Most continuous-delivery systems still intentionally deploy only the newest eligible desired state — `queue: max` makes durable queuing native, but that doesn't make queuing the right default for every deployment group. Decide per group whether "latest wins" or "every change ships" is the actual requirement before reaching for either option.

Build concurrency keys from bounded, trusted identifiers. Avoid a user-controlled value that can force unrelated deployments into the same group.

---

## 3. An Unshaped Matrix Multiplies Cost Faster Than Coverage

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

> **Edge case:** `exclude` only matters once the matrix has combinations nobody actually supports — the 3×2 matrix above would run 6 jobs without it; `exclude` drops the one unsupported pair down to 5.

Run the primary supported combination on every PR. Move the full compatibility matrix to merge queues, main, or a schedule if it is too expensive for fast feedback.

---

## 4. A Correct Cache Key Still Isn't a Trust Boundary

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

> **Production:** a cache is a performance optimization that also happens to be an input you don't fully control. Treat a cache hit as untrusted until it's cheap to validate.

Rules:

- a miss must still produce a correct build;
- never store credentials or signed release outputs;
- validate restored executables before privileged use;
- remember that branch cache scopes can expose default-branch caches to pull requests;
- rotate a cache namespace after suspected poisoning.

Use a registry-backed cache from **BuildKit** — Docker's build engine, which can push and pull reusable image layers to and from a container registry — for container layers, but publish the final image separately and identify it by digest. The BuildKit cache stores reusable layers for faster rebuilds, not the final release artifact; nothing should ever deploy straight from it.

---

## 5. Change Detection Speeds PRs but Can Hide a Broken Dependency

For a monorepo:

```text
detect changes
├── compute affected components
├── run their checks in a bounded matrix
├── include dependents of changed shared code
└── report one stable required gate
```

Add a scheduled full build to validate the dependency graph. Track cases where full builds fail but affected-only PR builds passed; they reveal gaps in change detection.

> **Edge case:** path filters on an entire *required* workflow can leave an expected check permanently pending — if the workflow never triggers, GitHub has nothing to report against the required-check name. Prefer an always-created detection job with conditional downstream jobs instead of filtering the whole workflow.

---

## 6. An Unbounded Wait Turns One Slow Dependency Into a Stuck Workflow

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

> **Core:** every wait needs an upper bound at both the job level and the command level — a hung dependency should fail loudly within minutes, not occupy a runner until the full `timeout-minutes` ceiling around it expires.

Retry only operations that are idempotent or have an idempotency key. Use exponential backoff with jitter for APIs. Do not retry deterministic compilation or assertion failures.

```text
safe retry:
    read request, idempotent PUT, digest-based deployment

unsafe without protection:
    charge card, append message, create unkeyed resource
```

---

## 7. Retention Policy Decides What You Can Debug Tomorrow

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

> **Production:** deleting a workflow run deletes its associated workflow artifacts — retention settings on the run don't protect anything you need past that run's own lifetime. Preserve release assets in a registry or release store with an explicit lifecycle policy instead.

---

## 8. Cleanup Is Reliable Only If You Verify the Delete

Preview environments and temporary accounts need two cleanup paths:

1. event-driven cleanup when a pull request closes;
2. scheduled reconciliation that deletes expired resources missed by events — a dropped webhook or a workflow that never reached its `if: closed` step leaves nothing behind to trigger cleanup otherwise.

Every resource should carry:

```text
repository = acme/orders
run_id     = 8912345678
pr_number  = 482
owner      = application-team
expires_at = 2026-07-31T12:00:00Z
```

Cleanup scripts must reject an empty identifier, verify ownership labels, and limit deletion to the expected scope. A deletion script with no success signal is trusted on faith: it can silently no-op against the wrong account, delete nothing because a filter typo excluded every candidate, or report success from a delete call that only queued the work.

> **Core:** a reconciliation pass works from a bounded, explicit candidate set — never "everything past a cursor" — and confirms each deletion afterward instead of trusting the delete call's `200 OK`.

One reconciliation pass, run at `2026-08-14T03:00:00Z` and scoped to `owner=application-team`:

```text
query: expires_at < now() AND repository = acme/orders

candidates:
  preview-482  owner=application-team  expires_at=2026-07-31T12:00:00Z  -> eligible
  preview-501  owner=application-team  expires_at=2026-08-10T09:00:00Z  -> eligible
  preview-517  owner=(missing)         expires_at=2026-08-01T00:00:00Z  -> rejected: no owner label
  preview-522  owner=billing-team      expires_at=2026-08-02T00:00:00Z  -> rejected: owner scope mismatch

delete: preview-482, preview-501
deleted_count = 2   (matches the 2 eligible candidates)

verify (re-query the same environments):
  preview-482 -> 404 Not Found
  preview-501 -> 404 Not Found
```

**Success signal:** `deleted_count` equals the number of eligible candidates, and a post-delete verification query returns not-found for every resource that was deleted.

⚠️ Silent failure looks like success: the delete call returns `200 OK`, but the resource is still present on the next reconciliation pass — a queued or asynchronous delete that never finished, or an IAM boundary that returns success on a no-op it silently blocked. Treat cleanup as unverified, and keep alerting, until the same identifier disappears from a follow-up existence check — not merely until the delete call returns without an error.

---

## 9. Runner and External Capacity Fail Together If Unmanaged

- Use `max-parallel` for large matrices.
- Use runner groups to constrain privileged workloads.
- Separate low-trust validation from deployment runners.
- Monitor job queue time and autoscaler convergence.
- Rate-limit calls to scanners, registries, and cloud APIs.
- Prefer ephemeral runners when self-hosting.
- Prebuild runner images for stable heavy toolchains, but patch them regularly.

> **Edge case:** warm persistent runners may reduce setup time but carry workspace and credential-contamination risk between jobs — a runner is only as trustworthy as the last job that ran on it. Performance cannot be evaluated separately from the trust model.

---

## 10. What Optimization Breaks If You're Not Careful

**Caching masks undeclared dependencies**

⚠️ A clean build fails on a cache miss even though every cached build passed. Run periodic cache-disabled builds and keep reconstruction complete.

**All matrices run on every documentation change**

⚠️ CI cost balloons because every combination runs on a change that touches no code. Compute affected scope and keep one scheduled full validation.

**Retries turn incidents into long queues**

⚠️ A struggling dependency gets hammered by every retrying job at once, deepening the incident. Bound attempts, classify errors, and stop retrying non-transient failures.

**Concurrency is assumed to be a release queue**

⚠️ Under the default `queue: single` behavior, a pending release can be silently replaced by a newer one and never deploy. Decide whether the desired state is "latest wins" — the default — or whether every change needs durable processing, in which case set `queue: max` on the group instead of reaching for an external queue.

**Optimization removes evidence**

⚠️ Faster feedback quietly drops the logs or artifacts needed to diagnose the next incident. Track escaped defects and full-build mismatches — faster feedback is useful only while risk remains controlled.

> **Key insight**: optimization always targets the *measured* critical path — never a guess at which job is slow — and every speed-up has to preserve the same trust, evidence, and bounded failure behavior the slower version had. A faster pipeline that silently drops verification, retries into an outage, or loses debug evidence isn't an optimization; it's a regression wearing a lower duration number.

---

## 11. References

- [Control workflow and job concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)
- [Dependency caching](https://docs.github.com/en/actions/concepts/workflows-and-actions/dependency-caching)
- [Workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts)
- [Self-hosted runners reference](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)

---

**Next**: [Permissions, Secrets, and OIDC](../03_security_and_supply_chain/01_permissions_secrets_and_oidc.md)
