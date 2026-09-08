# Testing and Quality Gates

> **Who this is for**: Engineers deciding what must pass before merge, promotion, and release.

## The short version

GitHub branch protection identifies a check by name and, when configured, its expected GitHub App source, but a real CI workflow has several jobs whose names or count can shift between runs. The fix is one aggregator job with a single fixed name that reads its upstream jobs' results; the ruleset deliberately requires that one check from GitHub Actions even though it could require several checks. Two upstream jobs plus that aggregator are enough to prove the merge-blocking mechanism end to end — every other test layer in this note plugs into the same aggregator later.

**What you need (3 things):**

1. Two independent check jobs in one workflow (here, `lint` and `unit`).
2. One aggregator job — named `required-ci` — that runs after them and reduces their results to a single pass/fail.
3. A branch protection ruleset that requires `required-ci` from the expected GitHub Actions App source, not merely any status with that name.

**The code:**

```yaml
name: ci
on: pull_request

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: echo "lint: 0 issues"     # stands in for a real linter

  unit:
    runs-on: ubuntu-latest
    steps:
      - run: echo "unit: 42 passed"    # stands in for a real test runner

  required-ci:
    name: required-ci
    needs: [lint, unit]
    if: ${{ always() }}
    runs-on: ubuntu-latest
    steps:
      - name: Reject every non-success result
        env:
          LINT_RESULT: ${{ needs.lint.result }}
          UNIT_RESULT: ${{ needs.unit.result }}
        run: |
          set -euo pipefail
          test "$LINT_RESULT" = success
          test "$UNIT_RESULT" = success
          echo "required-ci: lint and unit both succeeded"
```

**Success signal:** On a test PR, the merge box shows `required-ci` pending, then green once `lint` and `unit` both succeed. Force `unit` to exit 1: `required-ci` still runs because of `always()`, sees `UNIT_RESULT=failure`, exits non-zero, and turns red. Confirm the ruleset reports GitHub Actions as the expected source; a same-named result submitted by another integration must not satisfy it.

**Not handled yet:** [optional and matrix-shaped jobs](#3-harden-the-required-check-so-a-skipped-job-cannot-pass-it), [testing only the changed part of a monorepo](#4-directory-filters-miss-consumers-whose-own-path-did-not-change), [flaky-test policy](#5-a-flaky-test-is-a-defect-not-noise-to-rerun-away), [preview-environment cleanup](#7-cleanup-must-refuse-a-bad-identifier-before-it-deletes-anything).

---

This assumes you can already read a branch-protection ruleset and know what `CODEOWNERS` and a merge queue do; see [Branching and Production Change Control](02_branching_and_change_control.md) for those.

## 1. Most Test Layers Are Conditional; Three Run on Every PR

No single test suite proves release safety. Each layer finds a different class of defect at a different cost. A **unit test** exercises one function or class in isolation — cheap, and it catches wrong logic before anything is deployed, though a bug that only appears when two services talk to each other is outside what it can see. A **component test** runs a service against its real dependencies — an actual Postgres instance or Redis cache in CI, not a mock — and catches a broken query or serialization mismatch that a mocked dependency would hide. An **integration test** goes one step further and exercises deployed components together, including managed-service integrations, catching wiring problems no single service's tests would surface. A **contract test** checks that a producer's API and a consumer's expectation of it still agree, without deploying either side, and catches a breaking API change before the two services ever meet in a shared environment. A **synthetic journey** is a scripted, production-like user flow — log in, add to cart, check out — run continuously against the real system, and it catches degradation that only real infrastructure, real latency, and real data surface.

```text
                         ┌──────────────────────┐
                         │ Synthetic journeys   │  production-like, few
                    ┌────┴──────────────────────┴────┐
                    │ Integration and contract tests │
               ┌────┴────────────────────────────────┴────┐
               │ Component tests with real dependencies   │
          ┌────┴──────────────────────────────────────────┴────┐
          │ Unit tests, types, lint, policy, static analysis   │
          └────────────────────────────────────────────────────┘
             fast and numerous                  slow and focused
```

In practice, every PR runs three of these — format/lint, unit, and component tests (★ below); the rest are conditional on architecture, delivery stage, or release risk.

| Layer | Detects | Typical timing | On every PR? |
|-------|---------|----------------|--------------|
| **Format, lint, type checks** | Local consistency and invalid interfaces | Seconds; first on PR | **★ Always** |
| **Unit tests** | Logic and boundary behavior | Every PR | **★ Always** |
| **Component tests** | Service behavior with a database, queue, or cache | Every PR or affected paths | **★ Usually** |
| Contract tests | Producer/consumer compatibility | PR and package publication | Conditional — services with external consumers |
| Integration tests | Deployed components and managed-service integration | Staging or preview | Conditional — stage-gated |
| Smoke tests | Critical endpoint availability | Every deployment | Conditional — stage-gated |
| Synthetic journeys | User-visible and business behavior | After rollout and continuously | Conditional — post-deploy |
| Performance/resilience | Capacity, latency, timeout, and degradation behavior | Scheduled or before risky releases | Conditional — scheduled |

The `lint` and `unit` jobs in the short version above are exactly two of the three starred layers — the minimum default portfolio every PR runs regardless of architecture.

> **Key insight**: Promote only when the next environment will add new evidence. Repeating identical unit tests in five stages is not defense in depth.

---

## 2. Each Delivery Stage Must Add Evidence the Last One Could Not

Every stage below should test something the previous one could not — new infrastructure, new data, or new traffic — not repeat what already passed.

### Pull request

- formatting and linting;
- compile or type checking;
- unit and fast component tests;
- dependency review and secret scanning;
- static application security testing;
- container build validation;
- infrastructure formatting, validation, policy, and plan;
- migration lint and backward-compatibility tests.

### Main-branch build

- repeat tests that protect artifact integrity;
- build the immutable binary or image;
- scan the produced artifact;
- generate an **SBOM** — a Software Bill of Materials, a manifest listing every package and version compiled into the artifact — and **provenance** — signed, attestable metadata recording who built the artifact, from what source commit, and with which pipeline, so a consumer can verify origin instead of trusting a label;
- publish only after all build jobs pass.

### Staging

- deploy the exact candidate artifact;
- run integration and contract suites;
- verify migrations, external dependencies, and permissions;
- execute smoke and synthetic journeys;
- inspect operational signals.

### Production

- platform convergence and health checks;
- limited synthetic tests that are safe against real data;
- error, latency, saturation, and business-metric gates;
- rollback or halt when thresholds fail.

---

## 3. Harden the Required Check So a Skipped Job Cannot Pass It

> **Core:** branch protection can require exactly one job by name; everything below is about making that one job trustworthy once the graph behind it stops being two simple jobs.

The baseline aggregator in the short version has a blind spot. If `lint` fails, GitHub skips `required-ci` outright — its `needs` dependency didn't succeed, so it never runs. The PR still can't merge (a skipped required check does not satisfy the ruleset), but `required-ci` produces no logs and no explicit reason; engineers just see a blocked merge box. Real workflows make this worse with jobs that are legitimately optional, and with **matrix jobs** — one job definition that runs once per entry in a list of inputs, each producing its own check name such as `test (3.11)` and `test (3.12)` — whose names and count change release to release, so pointing branch protection at all of them breaks on the next dependency bump.

> **Production:** the change below is what you need before pointing this at a real workflow with optional or matrix-shaped jobs; skip it while you're still learning the baseline mechanism.

Two changes fix the blind spot: add `if: always()` so `required-ci` always runs and can inspect *why* its inputs failed, and read every upstream job's `result` explicitly instead of trusting the run to fail loudly on its own.

```yaml
name: ci
on: pull_request

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: echo "lint: 0 issues"

  unit:
    runs-on: ubuntu-latest
    steps:
      - run: echo "unit: 42 passed"

  integration:
    if: ${{ !contains(github.event.pull_request.labels.*.name, 'skip-integration') }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "integration: 12 passed"

  required-ci:
    name: required-ci
    if: ${{ always() }}
    needs: [lint, unit, integration]
    runs-on: ubuntu-latest
    steps:
      - name: Evaluate upstream jobs
        env:
          LINT_RESULT: ${{ needs.lint.result }}
          UNIT_RESULT: ${{ needs.unit.result }}
          INTEGRATION_RESULT: ${{ needs.integration.result }}
        run: |
          set -euo pipefail

          for result in "$LINT_RESULT" "$UNIT_RESULT"; do
            test "$result" = "success"
          done

          # integration is the only job allowed to be skipped — its own
          # `if:` condition, not this aggregator, decides whether that's OK
          case "$INTEGRATION_RESULT" in
            success|skipped) ;;
            *) exit 1 ;;
          esac
```

Configure the ruleset to require `required-ci`, not `lint`, `unit`, `integration`, or any matrix child individually.

That instruction spans two owners. The repository maintainer owns the workflow name; a GitHub repository or organization administrator owns the effective ruleset. Inspect the active rulesets targeting `main` and require this entry:

```json
{"context":"required-ci","integration_id":15368}
```

Here `integration_id` identifies the expected GitHub App; obtain the live value from the check/ruleset API rather than copying the illustrative number. A same-named status from a different source is a failure tell, not a pass.

Here's why the `case` statement is written this narrowly rather than accepting `skipped` for anything. Suppose someone later refactors `integration`'s `if:` condition and introduces a typo — the label match now always evaluates false. `integration` reports `needs.integration.result == "skipped"` on every PR, including ones that never asked to skip it, and nothing looks wrong because `skipped` is already an accepted value for that one job. If the same statement had instead treated `skipped` as acceptable for `unit` too — a plausible copy-paste — a similar typo in `unit`'s own condition would silently remove real test execution from every PR while `required-ci` still printed success and GitHub merged the change untested.

⚠️ The decision about whether a skipped job is acceptable belongs in reviewed code, scoped to the one job that is genuinely optional. Do not silently treat every `skipped` result as success.

Not every job earns a place in `needs: [...]`. Add one only once it has an owner, a defined failure disposition — what happens when it goes red — and enough run history to trust its signal; an unreliable scanner or a brand-new smoke test blocks merges on noise instead of risk. Until those three are true, run it as an informational check or on a schedule, and promote it into the aggregator once it has earned blocking status.

Test this on a real pull request before trusting it elsewhere:

- **Passing case**: once `lint`, `unit`, and (if triggered) `integration` finish, the merge box lists `required-ci` with a green check and "All checks have passed" — that exact name, not the jobs behind it, is what the ruleset evaluates.
- **Failing case**: force `unit` to fail. `required-ci` still runs — `if: always()` guarantees it — evaluates `UNIT_RESULT=failure`, exits non-zero, and the merge box shows `required-ci` red with merging blocked.
- **Silent-failure tell**: if the ruleset's required-check name doesn't exactly match — the ruleset says `required-ci` but a rename left the job's `name:` field as `Required CI`, or the workflow's `on:` no longer triggers on the PR's base branch — the merge box shows `required-ci` pending indefinitely, labeled something like "Expected — waiting for status to be reported." GitHub is waiting for a status that will never arrive; that symptom means the name or trigger has drifted, not that the pipeline is slow.

---

## 4. Directory Filters Miss Consumers Whose Own Path Did Not Change

Monorepos benefit from change detection:

```text
changed paths
├── services/orders/** ─────> orders tests
├── services/billing/** ────> billing tests
├── shared/** ──────────────> all dependent services
└── infra/** ───────────────> affected infrastructure plans
```

Prefer an explicit dependency map over simple directory filters.

For example, the detection job can read this small manifest:

```yaml
consumers:
  shared: [orders, billing]
  orders: [orders]
  billing: [billing]
```

| Changed input | `orders` output | `billing` output |
|---|---:|---:|
| `shared/money.py` | `true` | `true` |
| `services/orders/api.py` | `true` | `false` |

The job emits those booleans through `$GITHUB_OUTPUT`; the two service jobs consume them in `if:` expressions. This makes the transitive selection reviewable instead of hiding it in directory glob behavior.

> **Edge case:** a shared library bump only trips its own directory filter. Without a dependency map, every consumer that imports it skips testing entirely until the break shows up in staging.

Use a detection job to emit service outputs, run conditional jobs, then let the stable final gate validate the outcomes. Keep a scheduled full build to catch mistakes in the dependency map.

---

## 5. A Flaky Test Is a Defect, Not Noise to Rerun Away

Automatic reruns can distinguish environmental noise from a persistent failure, but they can also hide regression probability.

```text
first failure
   ├── deterministic assertion ─> fail immediately
   └── known transient boundary ─> bounded retry + record flake
                                      │
                                      └── quarantine owner + expiry
```

A **flake budget** is the allowed count or rate of first-attempt test failures during a stated window. Start with the starred fields; add the rest as the program matures. A flake policy should define:

- **★ which tests may retry;**
- **★ maximum attempts;**
- **★ a metric and budget for first-attempt failures;**
- **★ an owner and remediation deadline;**
- whether the test remains blocking;
- the maximum allowed quarantine period.

❌ `pytest || pytest` makes an intermittent regression look green without producing structured evidence.

✅ A test runner plugin records retries, preserves the first failure, and fails when the flake budget is exceeded.

```yaml
window: 7d
maximum_first_attempt_failure_rate: 0.01
maximum_attempts: 2
owner: team-payments
remediate_within: 5d
```

If `payments_retry_test` fails first on 18 of 1,000 runs, the report shows `1.8% > 1.0%` and the policy check fails even if every retry passes.

---

## 6. The Pipeline Is Code Too, and Its Own Changes Need Review

Pipeline code changes production. Test it accordingly:

| Delivery asset | Useful test |
|----------------|-------------|
| Workflow YAML | Syntax/action linting and a controlled test repository |
| Reusable workflow | Contract test for each supported input combination |
| Container image | Start it, wait for health, call a representative endpoint |
| Terraform module | Validate, policy scan, plan, and isolated apply/destroy |
| Migration | Upgrade from a production-like prior schema and test old/new binaries |
| Rollback workflow | Game day or scheduled non-production recovery |
| OIDC policy | Positive and negative assumption tests |

A workflow that deploys to cloud infrastructure typically assumes a cloud role through **OIDC** — OpenID Connect, a protocol where GitHub issues the running workflow a short-lived signed token carrying claims such as `repo:org/repo:ref:refs/heads/main`, and the cloud provider's trust policy checks those claims before handing back temporary credentials. "Positive and negative assumption tests" means exactly that exchange, tested both ways: a run from an allowed branch should receive credentials, and a run from a fork or a disallowed ref should be refused.

This explanatory trust-policy excerpt makes the evaluated fields visible; the complete cloud policy is owned by [Permissions, Secrets, and OIDC](../03_security_and_supply_chain/01_permissions_secrets_and_oidc.md):

```json
{"iss":"https://token.actions.githubusercontent.com","aud":"sts.amazonaws.com","sub":"repo:acme/orders:ref:refs/heads/main"}
```

An assumption attempt with that exact issuer, audience, repository, and ref is allowed. Change only `sub` to `repo:acme/orders:pull_request` and the cloud security-token service returns access denied. If the cloud or runner settings cannot be queried, record them as `unknown`: repository YAML proves declared permissions and runner labels, not the effective OIDC trust or runner-group assignment.

Use the same evidence inventory whenever a gate crosses systems: list the repository artifact, each linked GitHub/cloud/runner control plane, its owner and authoritative query, current access limitations, confirmed facts, and hypotheses. Close the loop with an observed red check, refused credential, or actual runner assignment; never infer external configuration from repository silence.

Suppose a contributor edits the deployment workflow to add `permissions: id-token: write` where it wasn't needed before, or widens that trust condition so any branch — not only `main` — can assume the deploy role. The next PR opened from that branch now receives real production credentials through the OIDC exchange above. Or, instead, they change `runs-on` to a self-hosted runner label they control, and their workflow executes on hardware with whatever network access and cached credentials that runner already has. Neither change touches application code, so application tests catch nothing.

Changes to workflow permissions, OIDC trust, and runner selection deserve code-owner review even when application tests are unchanged — independent review, by someone other than the change's author, is what stops a contributor from approving their own privilege escalation.

---

## 7. Cleanup Must Refuse a Bad Identifier Before It Deletes Anything

Production-like evidence does not require copying unrestricted production data.

- Generate deterministic fixtures for logic tests.
- Use isolated schemas, namespaces, accounts, or ephemeral environments.
- Mask or synthesize sensitive datasets.
- Give every preview environment an owner, expiry, and cleanup workflow.
- Avoid sharing mutable test data between parallel jobs.
- Make cleanup run under `if: always()` and also schedule a stale-resource sweeper.

```yaml
- name: Delete preview namespace
  if: ${{ always() }}
  env:
    PREVIEW_ID: pr-${{ github.event.pull_request.number }}
  run: ./scripts/delete-preview.sh "$PREVIEW_ID"
```

The step above is only as safe as the script it calls. An empty `$PREVIEW_ID` — a workflow re-run outside a pull-request context, say — must not resolve to "delete everything":

```bash
#!/usr/bin/env bash
set -euo pipefail

id="${1:-}"

# An empty or malformed value here would otherwise expand into deleting
# every namespace matched by a bare `kubectl delete namespace`.
if [[ ! "$id" =~ ^pr-[0-9]+$ ]]; then
  echo "refusing to delete: '$id' is not a valid pr-<number> namespace" >&2
  exit 1
fi

echo "resolved namespace: $id"
echo "before delete:"
kubectl get ns -l app=preview -o name

kubectl delete namespace "$id" --ignore-not-found
echo "after delete:"
kubectl get ns -l app=preview -o name
```

Run it for PR #482 while an unrelated PR #501 preview is still open, and the observation looks like this:

```text
resolved namespace: pr-482
before delete:
namespace/pr-482
namespace/pr-501
after delete:
namespace/pr-501
```

`pr-482` is gone; `pr-501` — a namespace the script never touched — is still there. If every `pr-*` namespace disappears instead, or the script exits `0` on an empty identifier, the guard on the input is missing or wrong.

---

## 8. Gate Quality Decays Without Deliberate Maintenance

**More tests make feedback slower but not safer**

Classify tests by defect type. Delete duplicates, move expensive suites later, and keep the PR critical path focused.

**Security scans are informative but not gating**

Define severity, exploitability, age, and exception policies. A scanner with no disposition process becomes noise — the same boundary from [section 3](#3-harden-the-required-check-so-a-skipped-job-cannot-pass-it): it stays advisory until ownership and disposition exist.

**The test environment is permanently shared**

Parallel runs interfere and failures cannot be reproduced. Isolate state per run and make resource identity traceable to the workflow.

**Required checks are renamed casually**

Rulesets depend on check identity. Treat gate names as an external contract and migrate them deliberately.

---

## 9. References

- [About status checks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/about-status-checks)
- [Troubleshooting required status checks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/troubleshooting-required-status-checks)
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)

---

**Next**: [GitHub Actions Workflow Building Blocks](../02_github_actions/01_workflow_building_blocks.md)
