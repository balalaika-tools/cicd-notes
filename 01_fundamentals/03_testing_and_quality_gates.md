# Testing and Quality Gates

> **Who this is for**: Engineers deciding what must pass before merge, promotion, and release. Read [Branching and Production Change Control](02_branching_and_change_control.md) first.

---

## 1. Build a Risk-Based Test Portfolio

No single test suite proves release safety. Each layer finds a different class of defect at a different cost.

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

| Layer | Detects | Typical timing |
|-------|---------|----------------|
| Format, lint, type checks | Local consistency and invalid interfaces | Seconds; first on PR |
| Unit tests | Logic and boundary behavior | Every PR |
| Component tests | Service behavior with a database, queue, or cache | Every PR or affected paths |
| Contract tests | Producer/consumer compatibility | PR and package publication |
| Integration tests | Deployed components and managed-service integration | Staging or preview |
| Smoke tests | Critical endpoint availability | Every deployment |
| Synthetic journeys | User-visible and business behavior | After rollout and continuously |
| Performance/resilience | Capacity, latency, timeout, and degradation behavior | Scheduled or before risky releases |

> **Key insight**: Promote only when the next environment will add new evidence. Repeating identical unit tests in five stages is not defense in depth.

---

## 2. Define Gates by Stage

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
- generate an SBOM and provenance;
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

## 3. Make One Stable Required Check

Large workflows often have optional or matrix jobs. Protect the branch with a final gate whose name is stable:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/lint.sh

  unit:
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/unit.sh

  integration:
    if: ${{ !contains(github.event.pull_request.labels.*.name, 'skip-integration') }}
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/integration.sh

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

          # This job is intentionally optional, so skipped is acceptable.
          case "$INTEGRATION_RESULT" in
            success|skipped) ;;
            *) exit 1 ;;
          esac
```

Configure the ruleset to require `required-ci`, not every dynamic matrix child.

⚠️ The decision about whether a skipped job is acceptable belongs in reviewed code. Do not silently treat every `skipped` result as success.

---

## 4. Test Changed Scope without Creating Blind Spots

Monorepos benefit from change detection:

```text
changed paths
├── services/orders/** ─────> orders tests
├── services/billing/** ────> billing tests
├── shared/** ──────────────> all dependent services
└── infra/** ───────────────> affected infrastructure plans
```

Prefer an explicit dependency map over simple directory filters. A shared package change may require tests in consumers whose paths did not change.

Use a detection job to emit service outputs, run conditional jobs, then let the stable final gate validate the outcomes. Keep a scheduled full build to catch mistakes in the dependency map.

---

## 5. Treat Flaky Tests as Defects

Automatic reruns can distinguish environmental noise from a persistent failure, but they can also hide regression probability.

```text
first failure
   ├── deterministic assertion ─> fail immediately
   └── known transient boundary ─> bounded retry + record flake
                                      │
                                      └── quarantine owner + expiry
```

A flake policy should define:

- which tests may retry;
- maximum attempts;
- a metric for first-attempt failures;
- an owner and remediation deadline;
- whether the test remains blocking;
- the maximum allowed quarantine period.

❌ `pytest || pytest` makes an intermittent regression look green without producing structured evidence.

✅ A test runner plugin records retries, preserves the first failure, and fails when the flake budget is exceeded.

---

## 6. Test the Delivery Mechanism

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

Changes to workflow permissions, OIDC trust, and runner selection deserve code-owner review even when application tests are unchanged.

---

## 7. Manage Test Data and Environments

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

The cleanup script must validate the identifier and refuse broad or empty targets.

---

## 8. Common Failure Modes

**More tests make feedback slower but not safer**

Classify tests by defect type. Delete duplicates, move expensive suites later, and keep the PR critical path focused.

**Security scans are informative but not gating**

Define severity, exploitability, age, and exception policies. A scanner with no disposition process becomes noise.

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
