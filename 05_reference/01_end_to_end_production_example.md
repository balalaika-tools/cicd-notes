# End-to-End Production Example: Python, Docker, ECR, and ECS

> **Who this is for**: Engineers assembling the earlier patterns into a concrete service pipeline. Read [Verification, Observability, and Rollback](../04_delivery_operations/04_verification_observability_and_rollback.md) first.

---

## 1. Target Architecture

Assume:

- a Python API packaged as a Docker image;
- GitHub Actions for CI and delivery orchestration;
- Amazon ECR for images;
- Amazon ECS for staging and production;
- separate AWS deployment roles assumed through OIDC;
- GitHub environments for staging and production;
- one immutable image promoted by digest.

```text
developer
    │ pull request
    ▼
GitHub repository
├── ruleset + CODEOWNERS
├── required-ci
└── merge queue
        │ accepted commit
        ▼
GitHub Actions trusted build
├── test accepted source
├── OIDC → artifact-publisher role
├── build image once
├── push to ECR
├── record digest
└── generate SBOM + provenance
        │
        ├── OIDC → staging role → ECS staging
        │                         └── smoke/integration
        │
        └── production environment approval/policy
              └── OIDC → production role → ECS production
                                              ├── circuit breaker
                                              ├── service stability
                                              ├── smoke + metrics
                                              └── previous task revision retained
```

---

## 2. Repository Layout

```text
orders/
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── pr-ci.yml
│       ├── release.yml
│       ├── rollback.yml
│       └── reusable-deploy-ecs.yml
├── src/
├── tests/
├── migrations/
├── infra/
│   ├── artifact/
│   ├── staging/
│   └── production/
├── scripts/
│   ├── ci.sh
│   ├── register-task-definition.sh
│   ├── smoke.sh
│   └── verify-deployment.sh
├── Dockerfile
├── requirements.txt
└── requirements-dev.txt
```

Keep complex logic in versioned scripts so it can be tested outside Actions.

---

## 3. Repository and Environment Configuration

GitHub ruleset for `main`:

```text
pull request required
1+ approval
code-owner approval for workflows, infra, and migrations
stale approvals dismissed
required check: required-ci
merge queue enabled
force push and deletion blocked
bypass restricted and audited
```

GitHub environments:

| Environment | Source policy | Gate | AWS role |
|-------------|---------------|------|----------|
| `artifact-publish` | `main` only | Automated | `orders-artifact-publisher` |
| `staging` | `main` only | Automated | `orders-staging-deploy` |
| `production` | `main` or protected release refs | Independent approval or policy | `orders-production-deploy` |

Environment role ARNs and non-secret target names belong in environment variables:

```text
vars.AWS_ROLE_ARN
vars.AWS_REGION
vars.ECS_CLUSTER
vars.ECS_SERVICE
vars.PUBLIC_BASE_URL
```

Do not use the same deployment role across environments.

---

## 4. Pull-Request Workflow

Use the full workflow from [Production Pull-Request CI](../02_github_actions/02_pull_request_ci.md). Its contract is:

```text
pull_request or merge_group
├── read-only GITHUB_TOKEN
├── lint + type + dependency audit
├── Python compatibility matrix
├── production-container build and health test
└── stable required-ci result
```

Add service-specific contract tests, migration lint, and Terraform validation without changing the stable `required-ci` name.

For accepted commits, run the core unit and artifact-integrity tests again in the trusted release workflow. A required PR result is merge evidence; the build workflow should not trust a PR-produced container as a release artifact.

---

## 5. Trusted Build Job

This excerpt uses AWS OIDC, Docker Buildx, and ECR. Readable major tags are shown for actions whose current SHA will change; production adoption must resolve every action to a reviewed full commit SHA.

```yaml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: release-orders
  cancel-in-progress: false

env:
  IMAGE: 123456789012.dkr.ecr.eu-west-1.amazonaws.com/orders

jobs:
  build:
    name: Build release candidate
    environment: artifact-publish
    runs-on: ubuntu-latest
    timeout-minutes: 35
    permissions:
      contents: read
      id-token: write
      attestations: write
    outputs:
      image: ${{ steps.image.outputs.name }}
      digest: ${{ steps.image.outputs.digest }}

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4
        with:
          persist-credentials: false

      - uses: actions/setup-python@a26af69be951a213d495a4c3e4e4022e16d87065 # v5
        with:
          python-version: "3.12"
          cache: pip

      - name: Test the accepted source
        run: |
          set -euo pipefail
          python -m pip install --require-hashes -r requirements-dev.txt
          ./scripts/ci.sh

      - name: Assume the artifact-publisher role
        uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Log in to ECR without storing a password
        env:
          AWS_REGION: ${{ vars.AWS_REGION }}
          REGISTRY: 123456789012.dkr.ecr.eu-west-1.amazonaws.com
        run: |
          set -euo pipefail
          aws ecr get-login-password --region "$AWS_REGION" |
            docker login \
              --username AWS \
              --password-stdin \
              "$REGISTRY"

      - name: Build and push exactly one candidate
        id: image
        env:
          IMAGE: ${{ env.IMAGE }}
        run: |
          set -euo pipefail
          docker buildx create --name release-builder --use
          docker buildx build \
            --pull \
            --provenance=false \
            --target production \
            --tag "$IMAGE:git-$GITHUB_SHA" \
            --label "org.opencontainers.image.revision=$GITHUB_SHA" \
            --metadata-file build-metadata.json \
            --push \
            .

          digest="$(
            jq -er '."containerimage.digest"' build-metadata.json
          )"
          [[ "$digest" =~ ^sha256:[0-9a-f]{64}$ ]]
          echo "name=$IMAGE" >> "$GITHUB_OUTPUT"
          echo "digest=$digest" >> "$GITHUB_OUTPUT"

      - name: Generate the final-image SPDX SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ env.IMAGE }}@${{ steps.image.outputs.digest }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Generate GitHub build provenance
        uses: actions/attest@v4
        with:
          subject-name: ${{ env.IMAGE }}
          subject-digest: ${{ steps.image.outputs.digest }}
          push-to-registry: true

      - name: Attest the SBOM
        uses: actions/attest@v4
        with:
          subject-name: ${{ env.IMAGE }}
          subject-digest: ${{ steps.image.outputs.digest }}
          sbom-path: sbom.spdx.json
          push-to-registry: true

      - name: Create release manifest
        env:
          IMAGE: ${{ steps.image.outputs.name }}
          DIGEST: ${{ steps.image.outputs.digest }}
        run: |
          set -euo pipefail
          jq -n \
            --arg repository "$GITHUB_REPOSITORY" \
            --arg sha "$GITHUB_SHA" \
            --arg run_id "$GITHUB_RUN_ID" \
            --arg image "$IMAGE" \
            --arg digest "$DIGEST" \
            '{
              source: {
                repository: $repository,
                sha: $sha,
                run_id: $run_id
              },
              artifact: {
                image: $image,
                digest: $digest
              }
            }' > release-manifest.json

      - name: Upload release manifest for downstream orchestration
        uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4
        with:
          name: release-manifest
          path: release-manifest.json
          if-no-files-found: error
          retention-days: 30
```

Why `--provenance=false`? The example creates GitHub provenance explicitly. In a real design, BuildKit provenance may also be retained, but the team should understand and verify each attestation rather than accidentally producing multiple statements with unclear policy.

The artifact-publisher role needs only the ECR operations required to authenticate, upload layers, and publish the image. It does not deploy ECS.

---

## 6. Register a Digest-Specific Task Definition

The deployment script copies an approved task-definition family revision, changes only the selected container image, removes read-only response fields, and registers a new revision:

```bash
#!/usr/bin/env bash
set -euo pipefail

family="${1:?usage: register-task-definition.sh FAMILY CONTAINER IMAGE DIGEST}"
container_name="${2:?usage: register-task-definition.sh FAMILY CONTAINER IMAGE DIGEST}"
image="${3:?usage: register-task-definition.sh FAMILY CONTAINER IMAGE DIGEST}"
digest="${4:?usage: register-task-definition.sh FAMILY CONTAINER IMAGE DIGEST}"

[[ "$family" =~ ^[a-zA-Z0-9_-]+$ ]]
[[ "$container_name" =~ ^[a-zA-Z0-9_-]+$ ]]
[[ "$digest" =~ ^sha256:[0-9a-f]{64}$ ]]

aws ecs describe-task-definition \
  --task-definition "$family" \
  --query taskDefinition \
  --output json > current-task-definition.json

jq \
  --arg container "$container_name" \
  --arg image_ref "$image@$digest" \
  '
    del(
      .taskDefinitionArn,
      .revision,
      .status,
      .requiresAttributes,
      .compatibilities,
      .registeredAt,
      .registeredBy,
      .deregisteredAt
    )
    | .containerDefinitions |= map(
        if .name == $container
        then .image = $image_ref
        else .
        end
      )
  ' current-task-definition.json > next-task-definition.json

matches="$(
  jq --arg container "$container_name" \
    '[.containerDefinitions[] | select(.name == $container)] | length' \
    next-task-definition.json
)"
test "$matches" -eq 1

aws ecs register-task-definition \
  --cli-input-json file://next-task-definition.json \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text
```

This script prints the new task-definition ARN, which becomes the immutable deployment input.

---

## 7. Reusable ECS Deployment Workflow

```yaml
name: Deploy ECS

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image:
        required: true
        type: string
      digest:
        required: true
        type: string

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    environment:
      name: ${{ inputs.environment }}
      url: ${{ vars.PUBLIC_BASE_URL }}
    concurrency:
      group: ecs-${{ inputs.environment }}-${{ vars.ECS_SERVICE }}
      cancel-in-progress: false
    runs-on: ubuntu-latest
    timeout-minutes: 35
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4
        with:
          persist-credentials: false

      - name: Validate release identity
        env:
          IMAGE: ${{ inputs.image }}
          DIGEST: ${{ inputs.digest }}
        run: |
          set -euo pipefail
          test "$IMAGE" = "123456789012.dkr.ecr.eu-west-1.amazonaws.com/orders"
          [[ "$DIGEST" =~ ^sha256:[0-9a-f]{64}$ ]]

      - name: Assume environment deployment role
        uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Register task definition
        id: task
        env:
          IMAGE: ${{ inputs.image }}
          DIGEST: ${{ inputs.digest }}
        run: |
          set -euo pipefail
          task_definition_arn="$(
            ./scripts/register-task-definition.sh \
              "$ECS_SERVICE" \
              orders \
              "$IMAGE" \
              "$DIGEST"
          )"
          echo "arn=$task_definition_arn" >> "$GITHUB_OUTPUT"

      - name: Deploy and wait
        env:
          TASK_DEFINITION_ARN: ${{ steps.task.outputs.arn }}
        run: |
          set -euo pipefail
          aws ecs update-service \
            --cluster "$ECS_CLUSTER" \
            --service "$ECS_SERVICE" \
            --task-definition "$TASK_DEFINITION_ARN" \
            --output json > deployment.json

          aws ecs wait services-stable \
            --cluster "$ECS_CLUSTER" \
            --services "$ECS_SERVICE"

      - name: Verify expected release and critical path
        run: |
          ./scripts/smoke.sh \
            "$PUBLIC_BASE_URL" \
            "$GITHUB_SHA"
```

Values in the `vars` context are not automatically shell environment variables. Add an explicit job-level `env` mapping in the adopted workflow:

```yaml
env:
  AWS_REGION: ${{ vars.AWS_REGION }}
  ECS_CLUSTER: ${{ vars.ECS_CLUSTER }}
  ECS_SERVICE: ${{ vars.ECS_SERVICE }}
  PUBLIC_BASE_URL: ${{ vars.PUBLIC_BASE_URL }}
```

The separated snippet makes that boundary visible; without the mapping, shell references such as `$ECS_SERVICE` are unset.

---

## 8. Stage, Verify, and Promote

Add these jobs to `release.yml`:

```yaml
  staging:
    needs: build
    permissions:
      contents: read
      id-token: write
    uses: ./.github/workflows/reusable-deploy-ecs.yml
    with:
      environment: staging
      image: ${{ needs.build.outputs.image }}
      digest: ${{ needs.build.outputs.digest }}

  verify-staging:
    needs: [build, staging]
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4
      - env:
          EXPECTED_DIGEST: ${{ needs.build.outputs.digest }}
        run: ./scripts/verify-deployment.sh staging "$EXPECTED_DIGEST"

  production:
    needs: [build, verify-staging]
    permissions:
      contents: read
      id-token: write
    uses: ./.github/workflows/reusable-deploy-ecs.yml
    with:
      environment: production
      image: ${{ needs.build.outputs.image }}
      digest: ${{ needs.build.outputs.digest }}
```

The production environment blocks the called deployment job until its gate passes. The same digest is passed to both environments.

For an approval lasting longer than the workflow's practical lifetime, write a durable signed release record and use a separate promotion workflow.

---

## 9. ECS Failure Detection

Configure the service:

```json
{
  "deploymentCircuitBreaker": {
    "enable": true,
    "rollback": true
  }
}
```

Also configure CloudWatch alarms for application-level failure where appropriate. The circuit breaker can detect inability to reach steady state; alarms add service metrics that platform readiness may miss.

Emit ECS deployment state-change events to EventBridge and alert on `SERVICE_DEPLOYMENT_FAILED`.

Automated ECS rollback still requires a previously completed deployment. The CI/CD workflow must observe the final service state rather than assuming the update request succeeded.

---

## 10. Database Migration Integration

For an additive migration:

```text
build immutable image
    ↓
run dedicated staging migration
    ↓
deploy staging + verify
    ↓
production environment gate
    ↓
run dedicated production migration once
    ↓
deploy compatible application digest
```

Keep migration concurrency separate:

```yaml
concurrency:
  group: database-production-orders
  cancel-in-progress: false
```

For a long backfill, start a separately observable operation and block only the release stage that truly depends on its completion.

---

## 11. Recovery

Record after each completed deployment:

```json
{
  "environment": "production",
  "service": "orders",
  "new_task_definition": "orders:184",
  "new_digest": "sha256:4ae0...9c1d",
  "previous_task_definition": "orders:183",
  "previous_digest": "sha256:82bd...7af0",
  "migration_version": "2026_07_29_01",
  "workflow_run_id": "8912345678"
}
```

Recovery sequence:

1. Stop progressive traffic or disable the feature flag.
2. Confirm data compatibility with the previous revision.
3. Update ECS to the recorded previous task definition.
4. Wait for service stability.
5. Run the same bounded smoke verification.
6. Record the recovery deployment and incident.

Do not rebuild the previous commit.

---

## 12. Final Production Flow

```text
PR
├── required-ci
├── owner review
└── merge queue
     ↓
main
├── trusted tests
├── build one image
├── push ECR tag
├── capture digest
├── SBOM + provenance
└── staging
     ├── register digest-specific task definition
     ├── deploy with staging OIDC role
     ├── wait for stability
     └── smoke/integration/metrics
          ↓
production environment gate
├── deploy same digest with production role
├── circuit breaker + alarms
├── verify expected release
├── publish deployment marker
└── retain previous task definition and compatibility state
```

---

## 13. References

- [Configuring OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [Deploying to Amazon ECS](https://docs.github.com/en/actions/how-tos/deploy/deploy-to-third-party-platforms/amazon-elastic-container-service)
- [ECS rolling deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)
- [ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html)

---

**Next**: [Production-Readiness Checklist](02_production_readiness_checklist.md)
