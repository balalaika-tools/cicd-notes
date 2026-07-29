# Environments and Artifact Promotion

> **Who this is for**: Teams moving a tested artifact through staging and production. Read [SBOM, Provenance, and Artifact Attestations](../03_security_and_supply_chain/03_sbom_provenance_and_attestations.md) first.

---

## 1. An Environment Is a Policy Boundary

Create GitHub environments for meaningful deployment targets:

```text
preview
staging
production
```

An environment can define:

- permitted deployment branches and tags;
- required reviewers;
- prevention of self-review;
- wait timers;
- environment variables and secrets;
- custom deployment protection rules;
- whether administrators may bypass protection.

A job referencing an environment waits for its protection rules before it starts and before it can access environment secrets.

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://app.example.com
    runs-on: ubuntu-latest
```

Feature availability for private repositories and advanced protection rules depends on the GitHub plan. Verify plan support before making a protection rule the only control.

---

## 2. Promotion Moves an Immutable Identity

```text
build
└── ghcr.io/acme/orders@sha256:4ae0...9c1d
        │
        ├── deploy to staging
        │     └── verify
        │
        └── deploy same digest to production
              └── verify
```

The production job should not run `docker build`.

Promotion inputs:

```yaml
artifact:
  image: ghcr.io/acme/orders
  digest: sha256:4ae0...9c1d
source:
  repository: acme/orders
  commit: 8f31c2a7d9...
  build_run_id: 8912345678
```

Before promotion:

- validate identifier syntax;
- fetch registry metadata;
- verify provenance and expected source;
- confirm staging deployed and tested the same digest;
- check vulnerability and policy status;
- ensure the release is not revoked.

---

## 3. A Staged Promotion Workflow

```yaml
name: Promote Release

on:
  workflow_dispatch:
    inputs:
      image:
        required: true
        type: string
      digest:
        required: true
        type: string

permissions:
  contents: read

concurrency:
  group: promote-orders
  cancel-in-progress: false

jobs:
  validate-release:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Validate immutable identity
        env:
          IMAGE: ${{ inputs.image }}
          DIGEST: ${{ inputs.digest }}
          GH_TOKEN: ${{ github.token }}
        run: |
          set -euo pipefail
          test "$IMAGE" = "ghcr.io/acme/orders"
          [[ "$DIGEST" =~ ^sha256:[0-9a-f]{64}$ ]]
          gh attestation verify \
            "oci://$IMAGE@$DIGEST" \
            --repo acme/orders

  staging:
    needs: validate-release
    permissions:
      contents: read
      id-token: write
    uses: acme/platform-workflows/.github/workflows/deploy-ecs.yml@0123456789abcdef0123456789abcdef01234567
    with:
      environment: staging
      image: ${{ inputs.image }}
      digest: ${{ inputs.digest }}

  verify-staging:
    needs: staging
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - run: ./scripts/verify-environment.sh staging

  production:
    needs: verify-staging
    permissions:
      contents: read
      id-token: write
    uses: acme/platform-workflows/.github/workflows/deploy-ecs.yml@0123456789abcdef0123456789abcdef01234567
    with:
      environment: production
      image: ${{ inputs.image }}
      digest: ${{ inputs.digest }}
```

The central workflow SHA is illustrative and must be replaced by the approved immutable version. The reusable workflow's input contract must match the caller; this example focuses on orchestration rather than the provider-specific inputs shown in the earlier reusable-workflow guide.

The `production` environment supplies the approval or automated protection gate. Approval authorizes the already-identified release; it must not allow the artifact to change afterward.

---

## 4. Configure Staging and Production Differently on Purpose

```text
staging
├── automatic from accepted main builds
├── environment-specific deployment role
├── production-like networking and dependencies
├── synthetic or masked data
└── integration, migration, and smoke gates

production
├── only trusted release refs/workflows
├── independent deployment role
├── no self-approval for manual gates
├── no admin bypass or strictly audited bypass
├── serialized deployment
└── progressive rollout and automated health policy
```

Environment parity means the architecture and failure behavior are representative, not that every capacity or dataset is identical.

Document intentional differences:

| Dimension | Difference | Risk and compensation |
|-----------|------------|-----------------------|
| Scale | Staging has fewer tasks | Run scheduled load tests |
| Data | Synthetic or masked | Maintain representative distributions |
| External providers | Sandbox account | Test production permissions separately |
| Feature flags | Internal audience | Validate final flag state before full rollout |

---

## 5. Choose a Promotion Model

| Model | Flow | Best fit |
|-------|------|----------|
| Same workflow run | Build → staging → production | Simple repository, short approval delay |
| Separate promotion workflow | Durable release record → selected environment | Long-lived candidates, recovery, independent authorization |
| GitOps pull request | Update desired-state digest and reconcile | Multiple services/environments, strong audit separation |
| Automated policy promotion | Staging signals authorize production | High-frequency, mature observability |

If an approval may wait for days, avoid depending on ephemeral workspace state or short-lived workflow artifacts. Store the release manifest durably and revalidate it at promotion time.

---

## 6. Use Protection Rules for New Evidence

Useful gates:

- independent change approval;
- change-window or freeze policy;
- incident status;
- security or vulnerability policy;
- staging deployment and test result;
- observability health;
- service ownership;
- regulatory change record.

GitHub custom deployment protection rules can integrate GitHub Apps with observability or change-management systems. At the time of writing, this feature is in public preview and plan availability varies.

Avoid fixed wait timers unless the elapsed time itself provides evidence. A canary bake period tied to metrics is stronger than an arbitrary delay with no measurement.

---

## 7. Preview Environments

```text
pull request opened/updated
    ↓ deploy isolated preview
    ↓ post URL and expiry
    ↓ run UI/integration checks
pull request closed
    ↓ delete preview
scheduled sweeper
    ↓ delete expired orphaned previews
```

Preview controls:

- no production credentials or unrestricted network access;
- per-PR namespace/account/schema;
- sanitized data;
- resource limits and cost labels;
- predictable URL access control;
- explicit expiry;
- idempotent cleanup.

Do not let a forked pull request run arbitrary infrastructure code with a privileged preview role.

---

## 8. Common Failure Modes

**Environment secrets are used as the approval mechanism**

The job is gated, but another job or workflow can still assume the cloud role. Align GitHub environment policy with cloud OIDC trust.

**An approver selects the artifact from a free-form field**

Validate against a signed build record and present source, digest, evidence, and differences before approval.

**Staging rebuilds or retags by mutable name**

Record and verify the digest at every target.

**Admin bypass becomes the normal path**

Remove routine causes of delay, restrict bypass, require a reason, and review bypass events.

---

## 9. References

- [Deployments and environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [Managing environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
- [Reviewing deployments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/review-deployments)
- [Deploying with GitHub Actions](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments)

---

**Next**: [Deployment Strategies and Progressive Delivery](02_deployment_strategies.md)
