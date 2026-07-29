# Reusable Workflows and Actions

> **Who this is for**: Platform and service teams standardizing CI/CD across repositories. Read [Build, Publish, and Promote](03_build_publish_and_promote.md) first.

---

## 1. Choose the Correct Reuse Mechanism

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

Avoid turning the reusable workflow into a hidden application. It should coordinate explicit tools and expose a small, typed contract.

---

## 2. Define a Reusable Deployment Contract

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
      # Resolve this readable major tag to a verified full SHA in production.
      - name: Assume the environment deployment role
        uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ inputs.aws_role_arn }}
          aws-region: ${{ inputs.aws_region }}

      - name: Update the service
        env:
          ECS_CLUSTER: ${{ inputs.cluster }}
          ECS_SERVICE: ${{ inputs.service }}
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
          ECS_CLUSTER: ${{ inputs.cluster }}
          ECS_SERVICE: ${{ inputs.service }}
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

---

## 3. Understand the Caller Boundary

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

Even though ten levels are supported, shallow call graphs are easier to review and debug.

---

## 4. Design Inputs as a Versioned API

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

Do not accept arbitrary shell fragments. If extensibility is needed, expose a declared mode or put service-specific behavior before or after the reusable workflow in separate jobs.

---

## 5. Keep Secrets Narrow

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

## 6. Combine Reusable Workflows and Composite Actions

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

---

## 7. Version and Roll Out Central Workflows

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

Options:

| Reference | Benefit | Cost |
|-----------|---------|------|
| Full commit SHA | Immutable and reviewable | Update automation is required |
| Protected release tag | Human-friendly | A tag can move unless governance prevents it |
| Branch | Immediate central rollout | A central change can break every caller simultaneously |

Use Dependabot or another updater to open reviewed SHA-update pull requests. Include the human-readable release version in a comment.

---

## 8. Observe Adoption and Failures

Track:

- caller repository and workflow;
- central workflow SHA;
- result and duration;
- input schema version;
- deprecated input usage;
- deployment target and artifact digest.

GitHub Enterprise Cloud records `prepared_workflow_job` audit data with caller refs and SHAs. Combine this with repository search or dependency graph data to find consumers before a breaking change.

---

## 9. Common Failure Modes

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
