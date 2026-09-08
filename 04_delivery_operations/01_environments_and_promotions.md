# Environments and Artifact Promotion

> **Who this is for**: Teams moving a tested artifact through staging and production.

## The short version

A workflow finishing green only proves its tests passed — nothing about that says it should be allowed to deploy to production or read a production secret. A GitHub environment is the boundary that makes that distinction real: a named deployment target whose protection rules must pass before a job can start or read its secrets. Promotion is what happens after that boundary exists — moving one already-built, already-verified artifact identity through each target in turn, never rebuilding it along the way.

**What you need (4 things):**

1. A validated immutable **digest** — e.g. `sha256:4ae0...9c1d`, the artifact's exact content hash, never a mutable tag.
2. A GitHub environment per target (`staging`, `production`) with its protection rules configured.
3. Verified **provenance** — signed evidence of which workflow and commit built that digest — checked against the approved builder, not just the repository.
4. A staging deployment and passing verification for that same digest before production is allowed to see it.

**The code:**

```yaml
jobs:
  validate-release:
    steps:
      - run: |
          gh attestation verify "oci://$IMAGE@$DIGEST" \
            --repo acme/orders \
            --signer-workflow acme/orders/.github/workflows/build.yml@refs/heads/main
  staging:
    needs: validate-release
    uses: acme/platform-workflows/.github/workflows/deploy-ecs.yml@0123456789abcdef0123456789abcdef01234567
    with: { environment: staging, digest: ${{ inputs.digest }} }
  verify-staging:
    needs: staging
    steps:
      - run: ./scripts/verify-environment.sh staging
  production:
    needs: verify-staging
    uses: acme/platform-workflows/.github/workflows/deploy-ecs.yml@0123456789abcdef0123456789abcdef01234567
    with: { environment: production, digest: ${{ inputs.digest }} }
```

**Success signal:** `verify-staging` exits `0`, and `curl https://staging.app.example.com/version` returns `sha256:4ae0...9c1d`; once the `production` environment's approval clears, `curl https://app.example.com/version` returns that same digest — never a different one.

**Not handled yet:** [GitHub plan limits on protection rules](#1-passing-ci-does-not-grant-access-to-production), [provenance and signer policy](#3-one-validated-digest-deploys-to-both-staging-and-production), [preview environment isolation](#7-a-preview-environment-is-untrusted-code-with-infrastructure-access), and [recovering from a bad promoted release](04_verification_observability_and_rollback.md).

---

For background on how this artifact's **SBOM** (software bill of materials) — its software-component inventory — and its build provenance were generated and attested before it ever reached promotion, see [SBOM, Provenance, and Artifact Attestations](../03_security_and_supply_chain/03_sbom_provenance_and_attestations.md). It is not required to follow the mechanics below.

---

## 1. Passing CI Does Not Grant Access to Production

A workflow finishes green, and its `deploy` job carries `id-token: write` and a step that assumes a cloud role. Nothing about that job's success says which targets it should be allowed to reach: without a boundary specific to the target, the same permissions that pushed a scratch namespace can just as easily assume the role over the account holding customer data, and the same job that read a staging API key can read the production one — because nothing told GitHub these two runs are not equivalent. A GitHub environment is what closes that gap: a named deployment target that a job must declare, whose protection rules must pass before the job starts, and whose secrets only become visible to a job that has passed them.

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

The environment administrator should export and review the effective GitHub-owned configuration, for example:

```json
{
  "name": "production",
  "deployment_branch_policy": {"protected_branches": true, "custom_branch_policies": false},
  "protection_rules": [
    {"type": "required_reviewers", "reviewers": ["acme/release-managers"], "prevent_self_review": true},
    {"type": "branch_policy"},
    {"type": "custom", "app": "change-policy"}
  ],
  "can_admins_bypass": false
}
```

The workflow proves only `environment: production`; this response proves allowed refs, reviewers, self-review, admin bypass, custom protection Apps, and whether enforcement is active. Record inaccessible fields as `unknown`, assign the repository/environment administrator as owner, and compare the export with the reviewed baseline on a schedule. A test run from an unprotected ref must remain blocked and receive no production credential.

> **Production:** required reviewers is usually the first control a team reaches for, but its default behavior is narrower than the name suggests: it takes only **one** approval from up to six listed users or teams. GitHub does not ship an all-reviewers or N-of-M quorum mode here — a single approver among the six is sufficient, and a single compromised or careless one is enough to approve. If a release genuinely needs independent multi-party sign-off, enforce that with a [custom deployment protection rule](#6-protection-rules-should-gate-on-evidence-not-elapsed-time) backed by an external approval service that itself tracks and requires multiple distinct approvers ([Deployments and environments — required reviewers](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments#required-reviewers), checked 2026-08-14).

A job referencing an environment waits for its protection rules before it starts and before it can access environment secrets.

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://app.example.com
    runs-on: ubuntu-latest
```

> **Edge case:** feature availability for private repositories and advanced protection rules depends on the GitHub plan. Verify plan support before making a protection rule the only control.

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

> **Core:** this is the mechanism every promotion model in this file builds on — validate one digest once, deploy that same digest to every target in turn, verify at each one. Nothing downstream of `validate-release` is allowed to change which bytes get shipped.

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

A digest is the sha256 hash of the artifact's exact bytes — it proves the same content reaches every target unchanged. Provenance is a separate claim: signed evidence of which workflow, commit, and builder produced that digest — it proves where the bytes came from, not merely that they're identical everywhere. Promotion needs both: the digest to guarantee nothing changed in transit, and provenance to guarantee what didn't change was trustworthy to begin with.

Before promotion:

- validate identifier syntax;
- fetch registry metadata;
- verify provenance and expected source;
- confirm staging deployed and tested the same digest;
- check vulnerability and policy status;
- ensure the release is not revoked.

---

## 3. One Validated Digest Deploys to Both Staging and Production

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
      - name: Validate immutable identity and approved provenance
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
            --repo acme/orders \
            --signer-workflow acme/orders/.github/workflows/build.yml@refs/heads/main

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

`--repo` alone only scopes the *search* for attestations to `acme/orders` — it does not restrict which workflow inside that repository produced one, and the CLI documents repository scope as the minimum, not the whole policy. A renamed, stale, or unofficial workflow still living in `acme/orders` can publish an attestation that passes `--repo acme/orders` on its own. `--signer-workflow` pins verification to the one workflow, on the one ref, allowed to build this image ([gh attestation verify](https://cli.github.com/manual/gh_attestation_verify), checked 2026-08-14).

A run that behaves correctly leaves a specific trail: `validate-release` succeeds before either environment is touched; the staging job shows `Success`; and the Amazon **Elastic Container Service (ECS)** task definition — the AWS control-plane object whose image is resolved for running tasks — reports the same digest. `verify-staging` exits `0`; the production environment records reviewer and timestamp before `Success`; and its task definition again resolves to that digest.

⚠️ The most common silent failure here is not a rejected deployment — it's an *unenforced* gate that nobody notices because every run still looks green. If the `production` environment has no required reviewers configured (removed, or never added when the environment was created), the `production` job starts the instant `verify-staging` finishes: there is no "Waiting for review" step in the run timeline, and the environment's deployment history shows no reviewer entry — but the workflow still reports `Success`, so nothing in the usual run summary flags that the approval never happened. Check the **Environments** tab's protection-rule list for the target itself, not just the workflow's pass/fail status.

> **Key insight**: an approval authorizes an already-validated artifact identity — the digest checked in `validate-release` — not a decision to pick or rebuild what actually gets deployed. A control that lets anything downstream of the approval change which bytes get shipped hasn't been strengthened by adding more reviewers to it.

---

## 4. Environment Parity Means Representative Behavior, Not Identical Infrastructure

> **Production:** the differences below are what you configure once staging and production environments already exist; treat this as pre-ship hardening rather than something to internalize on a first pass.

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

Configuration and feature-flag values are part of what differs by design between staging and production — see [Configuration Versioning and Recovery](05_configuration_versioning_and_recovery.md) for versioning, validating, and promoting that difference deliberately instead of letting it drift.

Document intentional differences:

| Dimension | Difference | Risk and compensation |
|-----------|------------|-----------------------|
| Scale | Staging has fewer tasks | Run scheduled load tests |
| Data | Synthetic or masked | Maintain representative distributions |
| External providers | Sandbox account | Test production permissions separately |
| Feature flags | Internal audience | Validate final flag state before full rollout |

---

## 5. The Promotion Model Should Match How Long a Release Waits to Deploy

| Model | Flow | Best fit |
|-------|------|----------|
| Same workflow run | Build → staging → production | Simple repository, short approval delay |
| Separate promotion workflow | Durable release record → selected environment | Long-lived candidates, recovery, independent authorization |
| GitOps pull request | Update desired-state digest and reconcile | Multiple services/environments, strong audit separation |
| Automated policy promotion | Staging signals authorize production | High-frequency, mature observability |

**GitOps** here means a reconciler — an operator or controller running separately from the pull request itself — continuously applies the repository's declared desired state to the live target. Merging the pull request that changes the digest is the *request*; it deploys nothing by itself. Promotion only happens once the reconciler's next sync picks up that change, which is why a GitOps promotion's success signal is the reconciler's convergence status, not the merge.

If an approval may wait for days, avoid depending on ephemeral workspace state or short-lived workflow artifacts. Store the release manifest durably and revalidate it at promotion time.

---

## 6. Protection Rules Should Gate on Evidence, Not Elapsed Time

Start with the three starred gates for the baseline; add conditional gates only when their evidence exists. Useful gates:

- **★ independent change approval;**
- change-window or freeze policy;
- incident status;
- security or vulnerability policy;
- **★ staging deployment and test result;**
- **★ observability health;**
- service ownership;
- regulatory change record.

A custom protection App receives `{environment:"production", digest:"sha256:4ae0...9c1d", staging_run:8912}`. It queries the staging result and error-rate window, then returns `approved` only when both match that digest and policy; otherwise it returns `rejected` with the evidence URL. GitHub shows the deployment job as waiting while the decision is pending and failed when rejected. Custom deployment protection rules have plan-dependent limits for private repositories, so confirm plan support before making one the sole gate.

Avoid fixed wait timers unless the elapsed time itself provides evidence. A **canary** — routing a small slice of traffic or instances to the new version while the rest keep serving the old one — paired with a **bake period** — a fixed window afterward spent only observing that slice's health before continuing further — is a stronger gate than an arbitrary delay with no measurement: the wait is doing something, not just passing time.

---

## 7. A Preview Environment Is Untrusted Code With Infrastructure Access

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

> **Production:** a preview is reachable by anything that can open a pull request, including forks you don't control. Every control below is required before that's true of your repository, not optional hardening.

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

## 8. These Four Mistakes Turn the Approval Gate Into Theater

**Environment secrets are used as the approval mechanism**

The job is gated, but another job or workflow can still assume the cloud role. Align GitHub environment policy with the cloud's OpenID Connect (**OIDC**) trust policy: OIDC is the federated identity assertion GitHub issues for a job, and it's that assertion — not the environment gate — that the cloud provider's trust policy actually evaluates before handing out credentials. If any workflow's trust policy accepts the claims a non-gated job can present just as easily, the environment's approval step is decorative.

**An approver selects the artifact from a free-form field**

Validate against a signed build record and present source, digest, evidence, and differences before approval.

**Staging rebuilds or retags by mutable name**

Record and verify the digest at every target.

> **Edge case:** admin bypass exists for genuine emergencies. Treat any use outside of one as a signal that the gate itself needs fixing, not as a feature to lean on routinely.

**Admin bypass becomes the normal path**

Remove routine causes of delay, restrict bypass, require a reason, and review bypass events.

---

## 9. References

- [Deployments and environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [Managing environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
- [Reviewing deployments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/review-deployments)
- [Deploying with GitHub Actions](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments)
- [gh attestation verify](https://cli.github.com/manual/gh_attestation_verify)

---

**Next**: [Deployment Strategies and Progressive Delivery](02_deployment_strategies.md)
