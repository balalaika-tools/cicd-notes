# Production Pull-Request CI

> **Who this is for**: Teams implementing fast, secure, and enforceable pull-request checks. Read [Workflow Building Blocks](01_workflow_building_blocks.md) first.

---

## 1. A Production-Oriented Python Workflow

This example keeps the default token read-only, supports merge queues, cancels stale PR runs, uses locked dependencies, validates an image, and exposes one stable ruleset check.

```yaml
name: Required CI

on:
  pull_request:
    branches: [main]
  merge_group:
    types: [checks_requested]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

defaults:
  run:
    shell: bash

jobs:
  static-analysis:
    name: Static analysis
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4

      - uses: actions/setup-python@a26af69be951a213d495a4c3e4e4022e16d87065 # v5
        with:
          python-version: "3.12"
          cache: pip

      - name: Install locked development dependencies
        run: python -m pip install --require-hashes -r requirements-dev.txt

      - name: Lint and type-check
        run: |
          set -euo pipefail
          python -m ruff format --check .
          python -m ruff check .
          python -m mypy src tests

      - name: Scan Python dependencies
        run: python -m pip_audit --require-hashes -r requirements.txt

  test:
    name: Python ${{ matrix.python }}
    runs-on: ubuntu-latest
    timeout-minutes: 20
    strategy:
      fail-fast: false
      matrix:
        python: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4

      - uses: actions/setup-python@a26af69be951a213d495a4c3e4e4022e16d87065 # v5
        with:
          python-version: ${{ matrix.python }}
          cache: pip

      - name: Install locked development dependencies
        run: python -m pip install --require-hashes -r requirements-dev.txt

      - name: Run tests
        run: |
          set -euo pipefail
          python -m pytest \
            --strict-markers \
            --junitxml="test-results/python-${{ matrix.python }}.xml"

      - name: Upload test evidence
        if: ${{ always() }}
        uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4
        with:
          name: test-results-python-${{ matrix.python }}
          path: test-results/
          if-no-files-found: error
          retention-days: 14

  container:
    name: Container validation
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4

      - name: Build the production target
        run: |
          set -euo pipefail
          docker build \
            --pull \
            --target production \
            --tag "orders-ci:${GITHUB_SHA}" \
            .

      - name: Start and verify the image
        run: |
          set -euo pipefail
          container_id="$(
            docker run \
              --detach \
              --publish 127.0.0.1:8080:8080 \
              "orders-ci:${GITHUB_SHA}"
          )"
          trap 'docker logs "$container_id"; docker rm --force "$container_id"' EXIT

          curl \
            --fail \
            --silent \
            --show-error \
            --retry 12 \
            --retry-all-errors \
            --retry-delay 2 \
            --max-time 5 \
            http://127.0.0.1:8080/health/ready

  required-ci:
    name: required-ci
    if: ${{ always() }}
    needs: [static-analysis, test, container]
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - name: Require every validation job
        env:
          STATIC_RESULT: ${{ needs.static-analysis.result }}
          TEST_RESULT: ${{ needs.test.result }}
          CONTAINER_RESULT: ${{ needs.container.result }}
        run: |
          set -euo pipefail
          test "$STATIC_RESULT" = "success"
          test "$TEST_RESULT" = "success"
          test "$CONTAINER_RESULT" = "success"
```

Configure `required-ci` as the ruleset's required status check.

---

## 2. Order Checks for Fast Failure

GitHub jobs normally start in parallel, which minimizes wall-clock time but may spend compute on a build after lint has already failed.

Choose explicitly:

```text
lowest latency:
    lint ─┐
    unit ─┼── all start together
    build ┘

lowest wasted compute:
    lint → unit → build

balanced:
    lint ───────┐
    unit ───────┼→ build or integration
    policy ─────┘
```

For most teams, cheap independent checks in parallel followed by expensive integration work is a good compromise.

---

## 3. Handle Forked Pull Requests Safely

Code from a fork is untrusted. The normal `pull_request` event is designed for validation with restricted access.

```yaml
on:
  pull_request:

permissions:
  contents: read
```

- Do not require cloud credentials for normal PR validation.
- Do not expose repository or environment secrets.
- Do not send untrusted builds to persistent privileged runners.
- Do not publish fork-built artifacts as trusted releases.
- Treat caches and uploaded artifacts from low-trust runs as untrusted.

If a PR needs a preview environment, use a reviewed approval boundary or a separate privileged workflow that consumes a narrowly validated artifact and does not execute scripts from it.

---

## 4. Avoid the `pull_request_target` Trap

`pull_request_target` is useful for labeling, commenting, or assigning reviewers because the workflow comes from the target branch and can have target-repository privileges.

```yaml
name: Label pull request

on:
  pull_request_target:
    types: [opened, edited]

permissions:
  contents: read
  pull-requests: write

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      # No checkout of the contributor's branch.
      - name: Apply a label through the API
        env:
          GH_TOKEN: ${{ github.token }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: gh pr edit "$PR_NUMBER" --add-label "needs-review"
```

❌ Checking out `github.event.pull_request.head.sha` and running its build scripts in this workflow can turn a pull request into privileged code execution.

---

## 5. Add Security and Policy Evidence

Typical PR checks include:

| Check | What it protects |
|-------|------------------|
| Secret scanning | Credentials committed to source |
| Dependency review | New vulnerable or policy-prohibited dependencies |
| SAST | Unsafe source patterns and data flows |
| IaC scanning | Public exposure, weak encryption, broad IAM |
| Container build | Invalid Dockerfile and missing runtime assets |
| Container scan | Known issues in OS and language packages |
| License policy | Incompatible dependency licenses |
| Migration lint | Destructive or non-compatible schema changes |

Define a disposition policy. For example: block reachable critical/high findings, require an expiry on accepted risk, and avoid blocking on untriaged informational output.

---

## 6. Keep Required Check Names Stable

These can accidentally break a ruleset:

- renaming a workflow or job used as a required check;
- conditionally omitting the required job;
- relying on a matrix-generated name;
- using path filters on the entire required workflow;
- running duplicate workflows with the same check name.

Use the aggregator pattern and treat its name as a public API. Change it by temporarily requiring both old and new gates, validating the transition, and then removing the old one.

---

## 7. Pull-Request Artifact Rules

PR artifacts are evidence, not production releases:

```text
safe PR artifacts
├── test reports
├── coverage
├── screenshots
├── plans
└── untrusted image used only in an isolated preview

trusted release artifact
└── rebuilt from an accepted commit in a trusted main-branch workflow
    or promoted only after a deliberately secured untrusted-build design
```

Retain failure diagnostics long enough for investigation, but avoid uploading secrets, raw production data, or complete runner workspaces.

---

## 8. Common Failure Modes

**CI installs floating dependencies**

The same commit produces a different result next week. Lock direct and transitive dependencies and verify hashes where the ecosystem supports it.

**The test command behaves differently locally**

Put logic in versioned scripts or task-runner commands, and let Actions call those commands.

**A green retry hides a flake**

Preserve first-attempt evidence and track flake rate. Quarantine only with an owner and expiry.

**CI is required but routinely bypassed**

Reduce latency, stabilize false failures, and restrict bypass. A control that operators cannot trust will not survive urgent work.

---

## 9. References

- [Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
- [Securely using `pull_request_target`](https://docs.github.com/en/actions/reference/security/secure-use)
- [Managing a merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)
- [Dependency caching](https://docs.github.com/en/actions/concepts/workflows-and-actions/dependency-caching)

---

**Next**: [Build, Publish, and Promote](03_build_publish_and_promote.md)
