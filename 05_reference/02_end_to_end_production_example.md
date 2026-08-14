# End-to-End Production Example: Python, Docker, ECR, and ECS

<!-- length-justification: kept as one file. This is the collection's single canonical
end-to-end trace: the build, the per-environment trust boundaries, the deploy mechanism,
and the operational failure/migration/recovery paths all describe one running pipeline.
A reader debugging "what happens when this deployment fails" needs failure detection,
migration sequencing, and recovery in the same view as the workflow that produced the
deployment, not a link away. Splitting sections 9-13 into a companion file would also
require every future edit here to keep two files' cross-references in sync. -->

> **Who this is for**: Engineers assembling the earlier patterns into a concrete service
> pipeline. Read [Verification, Observability, and Rollback](../04_delivery_operations/04_verification_observability_and_rollback.md) and
> [Provisioning the Reference Platform](01_provisioning_the_reference_platform.md) first.

## The short version

Rebuild the image for staging and rebuild it again for production, and the two are different artifacts wearing the same tag — "tested" and "deployed" can silently diverge. The fix: **build the image exactly once, push it to a registry, and promote that same immutable digest through every environment.**

**What you need (3 things):**

1. A source commit that already passed required CI (tests, lint, container health check).
2. A registry returning a content-addressed digest for the pushed image — Amazon ECR (Elastic Container Registry, AWS's Docker registry).
3. Two or more deploy targets that accept "run this exact digest" — Amazon ECS (Elastic Container Service, AWS's container orchestrator) services for staging and production.

**The code:**

```bash
# 1. Build once, push once — the registry returns a content-addressed digest.
docker buildx build --push -t "$IMAGE:git-$GITHUB_SHA" .
digest=$(jq -r '."containerimage.digest"' build-metadata.json)   # sha256:4ae0...9c1d

# 2. Deploy staging by that digest, then check it's really live.
./scripts/register-task-definition.sh orders orders "$IMAGE" "$digest"   # -> orders:183
aws ecs update-service --cluster staging --service orders --task-definition orders:183
curl -fsS https://staging.orders.example.com/version | jq -r .digest   # => sha256:4ae0...9c1d

# 3. Promote the identical digest to production — no rebuild, no new tag.
aws ecs update-service --cluster production --service orders --task-definition orders:184
curl -fsS https://orders.example.com/version | jq -r .digest   # => sha256:4ae0...9c1d
```

**Success signal:** production's `/version` reports `"digest":"sha256:4ae0...9c1d"` — the exact digest staging already verified, with no rebuild in between.

**Not handled yet:** [OIDC trust per environment](#3-every-environment-gets-its-own-trust-boundary-not-just-its-own-name), [SBOM and build attestation](#5-the-trusted-build-job-produces-one-attested-sha-pinned-image), [automated failure detection](#9-ecs-can-roll-back-automatically-but-only-after-a-completed-deployment), [migrations ahead of the digest](#10-migrations-run-once-ahead-of-the-application-digest-that-needs-them), and [recovery](#11-recovery-restores-the-previous-task-definition-it-never-rebuilds-one).

---

## 1. One Image, Built Once, Verified Once, Promoted Everywhere by Digest

> **Core:** the whole pipeline rests on one guarantee: the bytes verified in staging are the exact bytes that run in production — proven by digest, never re-created from a tag.

Assume:

- a Python API packaged as a Docker image;
- GitHub Actions for CI and delivery orchestration;
- Amazon ECR (Elastic Container Registry — AWS's managed Docker registry) for images;
- Amazon ECS (Elastic Container Service — AWS's container orchestrator) for staging and production;
- separate AWS deployment roles assumed through OIDC (OpenID Connect — a workload-identity protocol that trades a short-lived, GitHub-signed token for temporary AWS credentials, with no long-lived secret stored anywhere);
- GitHub environments for staging and production;
- one immutable image promoted by digest.

For how this ECR repository, these ECS clusters and services, the three deployment roles, and the first task-definition revision actually get created, see [Provisioning the Reference Platform](01_provisioning_the_reference_platform.md) — this walkthrough starts from the point where all of it already exists.

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

Three repository-side controls gate every commit before any workflow runs: a **ruleset** (GitHub's branch-protection policy object — the conditions a commit must satisfy before it can reach `main`), **CODEOWNERS** (a file mapping paths to the accounts whose approval those paths require), and the **merge queue** (GitHub serializes accepted pull requests, re-validates each one against the current `main`, and merges only if it still passes — so two independently-approved PRs can't combine into a broken `main`).

The trusted build then produces two records alongside the image itself: an **SBOM** (software bill of materials — a machine-readable inventory of every package the image contains) and **provenance** (a signed claim about which workflow run, commit, and builder produced the image). This example's SBOM is written in **SPDX** (Software Package Data Exchange — the format `anchore/sbom-action` emits below); provenance uses GitHub's own build-attestation format.

---

## 2. Versioned Scripts Keep Deploy Logic Testable Outside Actions

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

## 3. Every Environment Gets Its Own Trust Boundary, Not Just Its Own Name

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

### The OIDC trust contract behind `vars.AWS_ROLE_ARN`

Each role above is only as narrow as its IAM trust policy. Every trust policy must satisfy two conditions on the token GitHub mints for that job — a wrong or missing condition either breaks the assumption outright or, worse, lets a job it shouldn't trust succeed:

| Role | `aud` condition | `sub` condition |
|---|---|---|
| `orders-artifact-publisher` | `sts.amazonaws.com` | `repo:acme/orders:environment:artifact-publish` |
| `orders-staging-deploy` | `sts.amazonaws.com` | `repo:acme/orders:environment:staging` |
| `orders-production-deploy` | `sts.amazonaws.com` | `repo:acme/orders:environment:production` |

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:acme/orders:environment:staging"
        }
      }
    }
  ]
}
```

Swap only the `sub` value for the other two roles; nothing else in the policy changes. The `aud` condition matters independently of `sub`: without it, a token minted for some other relying party could still be presented here, because `sub` alone doesn't prove the token was meant for AWS.

For repositories created after 15 July 2026, or that have opted in to immutable subject claims, GitHub issues `sub` with the owner's and repository's numeric database IDs baked in instead of their names — `repo:acme@9821/orders@552041:environment:staging`, not `repo:acme/orders:environment:staging`. A trust policy written against the name-based format never matches on an opted-in repository, and the failure is silent from the workflow's side: `configure-aws-credentials` reports a plain `AccessDenied` on `AssumeRoleWithWebIdentity`, with nothing to say the `sub` format was the mismatch. Verify which format a given repository issues before writing the condition. The managed way is the [`actions-oidc-debugger`](https://github.com/github/actions-oidc-debugger) action; a one-off inline check works too (needs `permissions: id-token: write`, remove it once confirmed):

```yaml
- name: Print the OIDC subject this job would present
  run: |
    token="$(curl -sS -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq -r .value)"
    echo "$token" | cut -d. -f2 | base64 -d 2>/dev/null | jq -r .sub
```

([Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws), [About security hardening with OpenID Connect](https://docs.github.com/en/actions/reference/security/oidc), checked 2026-08-14)

### Do not use the same deployment role across environments

Say `orders-staging-deploy` and `orders-production-deploy` were the same role instead of two roles with two `sub` conditions. A compromised dependency in a staging-only test step, or a malicious pull request that manages to execute code inside the `staging` job, now runs with credentials production also trusts — no separate approval, no separate audit trail, and no second `sub` condition standing in the way. The compromise reaches every permission `orders-production-deploy` holds the moment the staging job executes, because AWS has no way to know that *this particular* assumption of the role came from the "wrong" workflow.

Two roles, each trusted only by its own environment-bound `sub`, remove exactly that path: a staging compromise can assume `orders-staging-deploy` and touch staging's ECS service — and nothing else. Production stays reachable only through a token whose `sub` names the `production` environment, which the staging job's token never carries.

---

## 4. A Green PR Check Is Merge Evidence, Not a Release Artifact

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

## 5. The Trusted Build Job Produces One Attested, SHA-Pinned Image

This excerpt uses AWS OIDC, Docker Buildx, and ECR. Every third-party action below is pinned to the exact commit its release tag pointed to when reviewed, with the readable tag kept as a trailing comment — the same full-SHA pin already used for `actions/checkout` and `actions/setup-python`.

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
  queue: max

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
      artifact-metadata: write
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
        uses: aws-actions/configure-aws-credentials@61815dcd50bd041e203e49132bacad1fd04d2708 # v5.1.1
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
        uses: anchore/sbom-action@e22c389904149dbc22b58101806040fa8d37a610 # v0.24.0
        with:
          image: ${{ env.IMAGE }}@${{ steps.image.outputs.digest }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Generate GitHub build provenance
        uses: actions/attest@1e69f48acb82d1966a394da916b4c1698aa569d6 # v4.2.2
        with:
          subject-name: ${{ env.IMAGE }}
          subject-digest: ${{ steps.image.outputs.digest }}
          push-to-registry: true

      - name: Attest the SBOM
        uses: actions/attest@1e69f48acb82d1966a394da916b4c1698aa569d6 # v4.2.2
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

> **Production:** without `queue: max`, GitHub Actions keeps only one *pending* run per `concurrency.group` — if a run is already in progress and another is already queued behind it, a third push cancels that queued run outright, silently dropping its commit from ever releasing. `queue: max` queues every run instead, up to **100 pending runs per group**; past that bound GitHub starts canceling new runs rather than silently dropping older ones. ([Control workflow concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency), checked 2026-08-14)

`ci.sh` reruns the same lint, type-check, and dependency-audit commands as `required-ci` — its full contents belong to [Production Pull-Request CI](../02_github_actions/02_pull_request_ci.md), which owns that script. This job never trusts the PR's own container, only its own fresh build.

> **Production:** `actions/attest` needs GitHub-hosted attestation storage. Public repositories get it automatically. Private or internal repositories need a **GitHub Enterprise Cloud** plan — plain **GitHub Enterprise Server (GHES) is not supported**, on any plan, because attestation storage is a GitHub.com-hosted service GHES doesn't run. Teams on GHES, or without Enterprise Cloud, skip both `actions/attest` steps above and keep `sbom.spdx.json` plus `release-manifest.json` (below) as their build record, or self-host attestation with a tool like `cosign` against their own transparency log. ([actions/attest#readme](https://github.com/actions/attest#readme), checked 2026-08-14)

> **Production:** both `actions/attest` calls set `push-to-registry: true`, which defaults `create-storage-record` to `true` — creating that storage record needs the `artifact-metadata: write` permission already added to this job's `permissions` block above. Without it, the step fails after the attestation is already signed and pushed, not before. Setting `create-storage-record: false` explicitly is the alternative if the linked artifact record isn't wanted. ([actions/attest#usage](https://github.com/actions/attest#usage), checked 2026-08-14)

Why `--provenance=false`? The example creates GitHub provenance explicitly. In a real design, BuildKit provenance may also be retained, but the team should understand and verify each attestation rather than accidentally producing multiple statements with unclear policy.

The artifact-publisher role needs only the ECR operations required to authenticate, upload layers, and publish the image. It does not deploy ECS.

---

## 6. A Script Turns an Approved Digest Into a New Task-Definition Revision

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

## 7. One Reusable Workflow Deploys Any Environment From the Same Inputs

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
        uses: aws-actions/configure-aws-credentials@61815dcd50bd041e203e49132bacad1fd04d2708 # v5.1.1
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

`smoke.sh` is the last step inside this reusable workflow — it runs once per environment, immediately after ECS reports steady state, using only the URL and commit SHA already in scope. It assumes the service exposes `GET /healthz` and `GET /version` (JSON: `status`, `commit`, `digest`):

```bash
#!/usr/bin/env bash
set -euo pipefail
# Confirms the newly deployed task is actually serving traffic under the expected
# commit — "steady state" only proves ECS accepted the task, not that requests succeed.
base_url="${1:?usage: smoke.sh BASE_URL EXPECTED_SHA}"
expected_sha="${2:?usage: smoke.sh BASE_URL EXPECTED_SHA}"

health="$(curl -fsS --max-time 10 "$base_url/healthz")"
test "$(jq -r '.status' <<<"$health")" = "ok"

version="$(curl -fsS --max-time 10 "$base_url/version")"
deployed_sha="$(jq -r '.commit' <<<"$version")"
test "$deployed_sha" = "$expected_sha"

echo "smoke check passed: commit $deployed_sha is live at $base_url"
```

---

## 8. The Same Digest Moves From Staging to Production Without a Rebuild

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
    environment: staging
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4
      - env:
          EXPECTED_DIGEST: ${{ needs.build.outputs.digest }}
          BASE_URL: ${{ vars.PUBLIC_BASE_URL }}
        run: ./scripts/verify-deployment.sh "$BASE_URL" "$EXPECTED_DIGEST"

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

`verify-staging` declares `environment: staging` for one reason: environment-scoped `vars` — including `PUBLIC_BASE_URL` — are only readable by a job that declares that environment. Staging's gate is already `Automated`, so this adds no extra approval, only the variable's visibility.

`verify-deployment.sh` is the orchestration-level check — independent of the smoke check that already ran inside the staging deploy job, so a compromised or buggy deploy job can't self-report success:

```bash
#!/usr/bin/env bash
set -euo pipefail
# Re-reads the running digest independently of the deploy job's own report.
base_url="${1:?usage: verify-deployment.sh BASE_URL EXPECTED_DIGEST}"
expected_digest="${2:?usage: verify-deployment.sh BASE_URL EXPECTED_DIGEST}"

[[ "$expected_digest" =~ ^sha256:[0-9a-f]{64}$ ]]

deployed_digest="$(curl -fsS --max-time 10 "$base_url/version" | jq -r .digest)"
test "$deployed_digest" = "$expected_digest"

echo "verified $base_url is serving $expected_digest"
```

The production environment blocks the called deployment job until its gate passes. The same digest is passed to both environments.

For an approval lasting longer than the workflow's practical lifetime, write a durable signed release record and use a separate promotion workflow.

### The whole release, traced end to end

Follow one push through every job above with concrete values:

1. `git push origin main` lands commit `a1b2c3d`.
2. **build** builds once, pushes `123456789012.dkr.ecr.eu-west-1.amazonaws.com/orders:git-a1b2c3d`, and outputs `digest=sha256:4ae0...9c1d`. The SBOM and both attestations are pushed alongside it.
3. **staging** calls `reusable-deploy-ecs.yml` with that image and digest. `register-task-definition.sh` registers a new revision and prints `arn:aws:ecs:eu-west-1:123456789012:task-definition/orders:183`; `update-service` plus `wait services-stable` return once ECS reports steady state; `smoke.sh` confirms `https://staging.orders.example.com/version` returns `{"status":"ok","commit":"a1b2c3d","digest":"sha256:4ae0...9c1d"}`.
4. **verify-staging** re-checks independently — `verify-deployment.sh` reads the same endpoint and confirms the digest still matches.
5. **production** calls the identical reusable workflow with the identical `image` and `digest` inputs. No new build runs. `register-task-definition.sh` prints `arn:aws:ecs:eu-west-1:123456789012:task-definition/orders:184`, referencing the exact same digest.
6. Final check: `https://orders.example.com/version` returns `{"status":"ok","commit":"a1b2c3d","digest":"sha256:4ae0...9c1d"}` — the digest staging verified, now serving production traffic.

**Success signal:** the `build` job's `digest` output, the digest inside both task definitions (`orders:183` and `orders:184`), and the digest both `/version` endpoints report are all the same 64-character hash. Two different task-definition revisions, two different ECS services, one identical set of bytes.

> **Key insight**: a production promotion changes exactly one thing — which verified, immutable digest the target's task definition points to. Authority, evidence, and recovery state don't travel with that digest: which role was trusted to deploy it, which approval or policy gated it, and what a rollback restores are all environment-specific and stay where they are.

---

## 9. ECS Can Roll Back Automatically, But Only After a Completed Deployment

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

## 10. Migrations Run Once, Ahead of the Application Digest That Needs Them

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
  queue: max
```

> **Production:** the same reasoning as [the release workflow's concurrency block](#5-the-trusted-build-job-produces-one-attested-sha-pinned-image) applies here — without `queue: max`, only one migration run stays queued at a time, so a third push while one migration runs and one is already pending cancels the pending one instead of running it later. `queue: max` queues up to 100 pending runs per group instead of silently dropping one. ([Control workflow concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency), checked 2026-08-14)

For a long backfill, start a separately observable operation and block only the release stage that truly depends on its completion.

---

## 11. Recovery Restores the Previous Task Definition, It Never Rebuilds One

Record after each completed deployment — this is the same production promotion traced in [§8](#8-the-same-digest-moves-from-staging-to-production-without-a-rebuild):

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

## 12. The Full Flow, Start to Finish

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

## 13. Where These Defaults Come From

- [Configuring OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [About security hardening with OpenID Connect](https://docs.github.com/en/actions/reference/security/oidc)
- [Deploying to Amazon ECS](https://docs.github.com/en/actions/how-tos/deploy/deploy-to-third-party-platforms/amazon-elastic-container-service)
- [ECS rolling deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)
- [ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html)
- [Control workflow concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)
- [actions/attest — usage and Enterprise availability](https://github.com/actions/attest#readme)

---

**Next**: [Production-Readiness Checklist](03_production_readiness_checklist.md)
