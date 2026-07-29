# GitHub Actions Workflow Building Blocks

> **Who this is for**: Engineers who understand CI/CD concepts and need a precise mental model for GitHub Actions. Start with [Testing and Quality Gates](../01_fundamentals/03_testing_and_quality_gates.md).

---

## 1. A Complete Minimal Workflow

```yaml
name: Pull Request CI

on:
  pull_request:
    branches: [main]
  merge_group:
    types: [checks_requested]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

defaults:
  run:
    shell: bash

jobs:
  test:
    name: test-python-${{ matrix.python }}
    runs-on: ubuntu-latest
    timeout-minutes: 20

    strategy:
      fail-fast: false
      matrix:
        python: ["3.11", "3.12", "3.13"]

    steps:
      - name: Check out the triggering revision
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4

      - name: Set up Python
        uses: actions/setup-python@a26af69be951a213d495a4c3e4e4022e16d87065 # v5
        with:
          python-version: ${{ matrix.python }}
          cache: pip

      - name: Install locked dependencies
        run: python -m pip install --require-hashes -r requirements-dev.txt

      - name: Run tests
        run: python -m pytest --junitxml=test-results/results.xml
```

The workflow declares six different concerns:

| Key | Concern |
|-----|---------|
| `on` | Which GitHub event creates a run |
| `permissions` | What the run's `GITHUB_TOKEN` may do |
| `concurrency` | Which runs may overlap |
| `jobs` | Independent execution units and their dependencies |
| `runs-on` | The runner trust and compute boundary |
| `steps` | Ordered commands or actions within one job |

---

## 2. Events Define the Security Context

The event determines more than timing. It determines the ref, payload, token behavior, and availability of secrets.

| Event | Common use | Important property |
|-------|------------|--------------------|
| `pull_request` | Validate proposed code | Forked code runs with restricted token and no normal secrets |
| `push` | Build accepted commits | Runs trusted repository code at the pushed ref |
| `merge_group` | Validate merge-queue candidates | Must be included when Actions supplies required queue checks |
| `workflow_dispatch` | Manual promotion or recovery | Validate caller-controlled inputs and selected ref |
| `workflow_call` | Reusable workflow contract | Runs inside the caller's workflow and permission ceiling |
| `repository_dispatch` | Custom or cross-repository event | Treat `client_payload` as untrusted input |
| `schedule` | Maintenance and full regression | Runs the default branch; schedules can be delayed under load |
| `pull_request_target` | Privileged PR metadata automation | Runs base-branch workflow with target-repository privileges |

⚠️ Do not check out and execute untrusted pull-request code in a privileged `pull_request_target` job. Use it only for carefully reviewed metadata operations, or split the work into an unprivileged producer and privileged consumer with a narrow artifact contract.

Event filters are conjunctive:

```yaml
on:
  pull_request:
    branches:
      - main
    paths:
      - "src/**"
      - "tests/**"
```

Both the branch and path conditions must match. A filtered workflow may never create a check, which matters when a ruleset requires that check.

---

## 3. Jobs Form a Directed Acyclic Graph

Jobs run in parallel unless `needs` creates an edge:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/lint.sh

  test:
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/test.sh

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/build.sh

  deploy-staging:
    needs: build
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/deploy.sh
```

Data does not automatically move with the graph:

- use job **outputs** for small strings such as a digest;
- use workflow **artifacts** for files and reports;
- publish release artifacts to a durable package or container registry;
- never use a cache as an artifact-transfer mechanism.

By default, a job whose dependency fails or is skipped is also skipped. Use `if: always()` only where the job explicitly interprets upstream results or performs cleanup.

---

## 4. Steps Share a Runner, Jobs Do Not

Steps in one job share the workspace and process environment files. Jobs may run on different machines.

```yaml
steps:
  - name: Produce a step output
    id: version
    run: |
      set -euo pipefail
      version="$(./scripts/version.sh)"
      echo "value=$version" >> "$GITHUB_OUTPUT"

  - name: Make a variable available to later steps in this job
    run: echo "RELEASE_VERSION=${{ steps.version.outputs.value }}" >> "$GITHUB_ENV"
```

Map a step output through a job to another job:

```yaml
jobs:
  build:
    outputs:
      digest: ${{ steps.image.outputs.digest }}
    steps:
      - id: image
        run: echo "digest=sha256:4ae0..." >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    steps:
      - env:
          IMAGE_DIGEST: ${{ needs.build.outputs.digest }}
        run: ./scripts/deploy.sh "$IMAGE_DIGEST"
```

Keep outputs non-sensitive. GitHub may redact values that look secret, and job outputs are visible to later workflow logic.

---

## 5. Expressions Are Evaluated before the Shell

`${{ ... }}` is GitHub's expression language, not shell syntax.

```yaml
# ❌ The PR title is inserted into a generated shell script.
- run: echo "${{ github.event.pull_request.title }}"

# ✅ Pass untrusted data as an environment variable, then quote it in the shell.
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    set -euo pipefail
    printf '%s\n' "$PR_TITLE"
```

Treat these as untrusted unless proven otherwise:

- branch and tag names;
- pull-request, issue, and commit titles or bodies;
- labels, usernames, email addresses, and repository names;
- `workflow_dispatch` and `repository_dispatch` inputs;
- artifact contents from less-trusted jobs.

Prefer a real script in the repository when logic grows beyond a few lines. It is easier to lint, test, and run locally.

---

## 6. Contexts and Configuration Boundaries

| Value | Use for | Avoid |
|-------|---------|-------|
| `github` context | Event metadata and run identity | Direct interpolation of attacker-controlled fields into `run` |
| `inputs` | Typed reusable/manual workflow parameters | Secrets |
| `vars` | Non-secret organization, repository, or environment configuration | Credentials |
| `secrets` | Sensitive values that cannot use federation | Control flow that reveals whether a secret exists |
| `env` | Process inputs within a known scope | Cross-workflow contracts |
| `needs` | Job results and outputs | Large data |
| `matrix` | Declared job variations | Unbounded, user-controlled fan-out |

Set static configuration at the narrowest useful scope. Set permissions and sensitive values at job scope so unrelated jobs never receive them.

---

## 7. Artifacts, Caches, and Registries

```text
dependency cache
    purpose: accelerate reconstruction
    correctness: build must work without it

workflow artifact
    purpose: preserve reports or pass files between jobs/runs
    lifetime: bounded by retention and workflow history

package/container registry
    purpose: durable, versioned release distribution
    identity: version plus immutable digest
```

Never cache secrets. Treat restored cache contents as untrusted input. Pin the cache key to lockfiles and relevant tool versions:

```yaml
- uses: actions/cache@v4 # Pin to a verified full commit SHA in production.
  with:
    path: ~/.cache/pip
    key: pip-${{ runner.os }}-${{ hashFiles('requirements*.txt') }}
    restore-keys: |
      pip-${{ runner.os }}-
```

Setup actions often implement dependency caching with less YAML, but the same trust rules apply.

---

## 8. Environments and Concurrency Are Independent

An environment controls deployment policy and secrets. A concurrency group controls overlap. Referencing `production` does not automatically serialize deployments.

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://app.example.com
    concurrency:
      group: deploy-production
      cancel-in-progress: false
    runs-on: ubuntu-latest
```

Configure both when the target cannot tolerate overlapping state changes. See [Environments and Promotions](../04_delivery_operations/01_environments_and_promotions.md) for protection rules.

---

## 9. Failure Semantics

Use these controls deliberately:

```yaml
jobs:
  integration:
    timeout-minutes: 30
    continue-on-error: false
    strategy:
      fail-fast: false
```

- `timeout-minutes` bounds a stuck job.
- `continue-on-error` changes how failure affects the workflow; it should not hide required evidence.
- matrix `fail-fast` cancels sibling matrix jobs after one failure.
- `if: failure()` runs when an earlier step or dependency failed.
- `if: cancelled()` distinguishes operator or concurrency cancellation.
- `if: always()` is appropriate for cleanup and explicit result aggregation.

External commands also need their own connect and operation timeouts. A job timeout is too coarse to explain which dependency stalled.

---

## 10. References

- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
- [Contexts reference](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- [Script injections](https://docs.github.com/en/actions/concepts/security/script-injections)
- [Workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts)

---

**Next**: [Production Pull-Request CI](02_pull_request_ci.md)
