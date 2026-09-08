# Verification, Observability, and Rollback

> **Who this is for**: Teams defining when a deployment is actually successful and how to recover when it is not.

## The short version

A deployment API call can return success while the new code never serves a single real request — the platform accepted the change, but instances never converged, a dependency is unreachable, or a load balancer is still routing to the previous release. Deployment success is not one boolean; it is a chain of evidence, and each link has to be checked, because passing one link never implies the next one holds.

**What you need (3 things):**

1. A readiness endpoint that reports the immutable `image_digest`, with commit retained only as diagnostic metadata.
2. The digest the pipeline promoted, so the check compares exact bytes rather than a source revision that may have been built more than once.
3. One request against a real dependency-backed path, since a process can be "ready" while its database or queue connection is broken.

**The code:**

```bash
#!/usr/bin/env bash
set -euo pipefail

base_url="${1:?usage: smoke.sh BASE_URL EXPECTED_DIGEST}"
expected_digest="${2:?usage: smoke.sh BASE_URL EXPECTED_DIGEST}"

health_json="$(curl --connect-timeout 5 --max-time 20 --fail --silent --show-error "$base_url/health/ready")"
actual_digest="$(jq -er '.image_digest' <<< "$health_json")"

if [[ "$actual_digest" != "$expected_digest" ]]; then
  echo "FAIL: /health/ready reports digest ${actual_digest}, expected ${expected_digest}" >&2
  echo "  (instance is healthy but running a different release)" >&2
  exit 1
fi

curl --connect-timeout 5 --max-time 20 --fail --silent --show-error "$base_url/api/v1/catalog?limit=1" > /dev/null
echo "OK: serving digest ${actual_digest}, representative request succeeded"
```

**Success signal:** `OK: serving digest <expected_digest>, representative request succeeded` on stdout, exit code `0`.

**Not handled yet:** [gating a progressive rollout on telemetry](#5-an-absolute-threshold-fails-at-normal-peak-and-passes-during-a-quiet-outage), [proving a rollback target is actually eligible](#7-a-rollback-that-trusts-its-own-input-can-redeploy-the-wrong-or-revoked-artifact), [recovering from a data-incompatible release](#6-recovery-is-a-decision-tree-not-a-single-rollback-button), and [measuring the delivery system without gaming it](#9-measure-the-system-not-the-people-running-it).

---

For the pipeline stages that produce the digest this note verifies, see [Infrastructure and Database Changes](03_infrastructure_and_database_changes.md).

---

## 1. Deployment Command Success Is Not Service Success

The short version's check gets you from "the API returned 200" to "the expected commit answered one request." A production deployment needs that same reasoning carried through every layer between the deploy API and a user's browser — each layer can look fine while the one above it is broken.

```text
deployment API accepted change
    ↓
platform converged
    ↓
instances are ready
    ↓
dependencies and critical paths work
    ↓
service-level signals remain healthy
    ↓
business behavior remains healthy
```

| Layer | Evidence |
|-------|----------|
| Platform | Desired/ready instance count, rollout state, task events |
| Process | Startup completed, no crash loop, expected version |
| Dependency | Database, queue, cache, and external API behavior |
| API | Representative authenticated and unauthenticated requests |
| Service | Error, latency, traffic, saturation, SLO burn |
| Business | Orders accepted, jobs completed, payments authorized |

Reconcile the same identity across owners instead of trusting the application alone:

```text
release manifest (pipeline owner)       expected digest = sha256:4ae0...9c1d
platform task/pod (platform owner)      resolved digest = sha256:4ae0...9c1d
serving /health response (service)      observed digest = sha256:4ae0...9c1d
trace for request smoke-8912 (telemetry) release.digest = sha256:4ae0...9c1d
decision                                MATCH
```

If any inspection surface is inaccessible, the decision is `unknown`, not success. The request-correlated trace proves the instance reporting the digest also handled the representative request.

A **service-level objective (SLO)** is the reliability target you've committed to for a signal — for example, 99.9% of requests succeeding within 300ms over a rolling 28 days. Its **burn rate** is how fast that commitment's failure budget is being spent: a burn rate of 10x means the whole month's allowed failure budget would be gone in about three days if the current error rate held. A deployment can look healthy on every layer above and still be burning SLO budget fast — the business layer just hasn't caught up to the damage yet.

---

## 2. A Health Check That Only Says OK Hides Which Release Is Running

```text
/health/live
    process is alive; should not fail for a temporary dependency outage

/health/ready
    instance can safely receive traffic

/health/startup
    initialization has completed
```

Return version metadata separately or in an authenticated diagnostic endpoint:

```json
{
  "status": "ready",
  "release": "2.7.0",
  "commit": "8f31c2a7d9...",
  "image_digest": "sha256:4ae0...9c1d"
}
```

Avoid leaking sensitive configuration. The pipeline should confirm it reached the expected digest, not merely any healthy version.

---

## 3. A Smoke Test That Fails Silently Teaches the Next Responder Nothing

The short version's script proves the concept with a single attempt. A smoke test that runs unattended immediately after a rollout needs to tolerate a few seconds of platform churn, and — when it fails — say exactly what it saw instead of leaving a bare non-zero exit code for someone to reverse-engineer later.

```bash
#!/usr/bin/env bash
set -euo pipefail

base_url="${1:?usage: smoke.sh BASE_URL EXPECTED_COMMIT}"
expected_commit="${2:?usage: smoke.sh BASE_URL EXPECTED_COMMIT}"

# --retry tolerates the platform still draining old tasks right after rollout;
# --max-time bounds each attempt so one hung task can't stall the whole gate
health_json="$(
  curl \
    --connect-timeout 5 \
    --max-time 20 \
    --retry 10 \
    --retry-all-errors \
    --retry-delay 3 \
    --fail \
    --silent \
    --show-error \
    "$base_url/health/ready"
)"

actual_commit="$(jq -er '.commit' <<< "$health_json")"
if [[ "$actual_commit" != "$expected_commit" ]]; then
  echo "SMOKE FAIL: /health/ready reports commit ${actual_commit}, expected ${expected_commit}" >&2
  echo "  (readiness succeeded for a different release -- stale instance, stuck rollout," >&2
  echo "  or a load balancer still routing to the previous task set)" >&2
  exit 1
fi
echo "SMOKE OK: serving expected commit ${actual_commit}"

curl \
  --connect-timeout 5 \
  --max-time 20 \
  --fail \
  --silent \
  --show-error \
  "$base_url/api/v1/catalog?limit=1" > /dev/null
echo "SMOKE OK: representative request (GET /api/v1/catalog) succeeded"
```

Production smoke tests must:

- avoid destructive or duplicate side effects;
- use dedicated synthetic identities and data;
- have timeouts;
- verify the expected release;
- produce diagnostic evidence without exposing secrets.

**Success signal:** both `SMOKE OK` lines on stdout. **Silent-failure tell:** a smoke test that skips the commit comparison and only checks HTTP status passes even when a stale instance answers — a `200` tells you a server responded, not which release it was running.

> **Core:** Sections 1 through 3 — the evidence-chain model, health endpoints that expose release identity, and a smoke test that checks it — are the baseline every reader needs. Everything from here on hardens telemetry, gating, and rollback around that baseline.

---

## 4. Without a Release Marker in Telemetry, an Incident Can't Be Traced to Its Deploy

Send a deployment event containing:

```json
{
  "service": "orders",
  "environment": "production",
  "release": "2.7.0",
  "commit": "8f31c2a7d9...",
  "digest": "sha256:4ae0...9c1d",
  "workflow_run_id": "8912345678",
  "deployment_id": "ecs-svc/1234567890",
  "actor": "cicd-app[bot]"
}
```

Here **ECS** means Amazon Elastic Container Service; the service deployment identifier lets an operator query rollout state, task failures, and the resolved task definition in AWS's control plane.

Add release identity to logs, traces, metrics, and error reports. During an incident, responders should be able to move from a latency spike to the deployment and then to its source and artifact.

A metric label needs a small, bounded set of possible values to stay cheap; commit and digest are **high-cardinality** — a label with enough distinct values to multiply the number of stored metric series (and their cost), since every deploy mints a new one. Avoid high-cardinality metric labels for commit or digest in every series. Use deployment events, logs, trace resource attributes, or a bounded release label appropriate to the telemetry backend.

> **Edge case:** high cardinality only bites once telemetry volume or cost is already a problem. Most services can defer this until a growing bill or a slow dashboard query flags it — just don't reach for a raw commit or digest label as the default.

---

## 5. An Absolute Threshold Fails at Normal Peak and Passes During a Quiet Outage

> **Production:** Sections 5 through 8 — SLO-aware gating, the recovery state machine, the rollback workflow, and what to retain per deployment — are required before this verification approach runs unattended against real production traffic. Skip them while you're still proving the baseline end to end.

An absolute threshold may fail during normal peak traffic or pass during a low-traffic outage. Compare:

- candidate against the current version;
- current window against historical baseline;
- fast and slow SLO burn windows;
- technical and business signals;
- minimum sample size.

```text
continue rollout when:
    candidate error rate <= policy
    candidate p95 latency <= policy
    SLO burn <= policy
    business invariant violations = 0
    sample size >= minimum

otherwise:
    halt traffic increase
    preserve evidence
    roll back or roll forward
```

**p95 latency** is the value below which 95% of requests fall — high enough to catch a real tail regression, without letting one slow outlier dominate the signal the way a raw maximum would.

Missing telemetry should stop a risky progressive rollout. An observability outage removes evidence; it does not prove the release healthy.

---

## 6. Recovery Is a Decision Tree, Not a Single Rollback Button

```text
deploying
   │
   ├── healthy ──────────────> completed
   │
   └── unhealthy
          ├── traffic only affected ─> disable flag / route back
          ├── binary compatible ─────> deploy previous digest
          ├── data incompatible ─────> corrective migration / roll forward
          └── unknown ───────────────> halt, contain, incident response
```

Automated rollback is appropriate when the signal is reliable and the operation is safe. Otherwise automate the halt and evidence capture, then require an operator decision.

---

## 7. A Rollback That Trusts Its Own Input Can Redeploy the Wrong or Revoked Artifact

Two things established earlier in this note constrain what a rollback workflow is allowed to do: the decision tree above only reaches "deploy previous digest" when the failure is binary-compatible, and [what you retain per deployment](#8-what-you-retain-per-deployment-is-what-rollback-can-later-prove) means every past deployment left a record of its digest and compatibility status. A rollback implementation that ignores both and takes an operator-supplied digest at face value reopens exactly the failure it exists to prevent.

Concretely: an operator fills in a digest during an incident, from memory or a chat message. If it's mistyped, or it's a digest that was itself revoked because it caused the original incident, or it's from three releases back and predates a schema migration the database has since applied, a workflow that only checks "is this a syntactically valid digest with a valid attestation" deploys it anyway. Attestation only proves *a* trusted build produced that digest — it says nothing about whether that digest is the right thing to run next.

```yaml
name: Roll Back Production

on:
  workflow_dispatch:
    inputs:
      incident:
        required: true
        type: string
      confirm_digest:
        description: "Digest you expect to roll back to, for confirmation only -- the workflow resolves the actual target itself"
        required: false
        type: string

permissions:
  contents: read

jobs:
  rollback:
    environment: production
    concurrency:
      group: deploy-production-orders
      cancel-in-progress: false
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - name: Validate incident reference
        env:
          INCIDENT: ${{ inputs.incident }}
        run: |
          set -euo pipefail
          [[ "$INCIDENT" =~ ^INC-[0-9]+$ ]]

      - name: Resolve rollback target from the deployment record
        id: resolve
        env:
          GH_TOKEN: ${{ github.token }}
          CONFIRM_DIGEST: ${{ inputs.confirm_digest }}
        run: |
          set -euo pipefail
          # Looks up the last known-good digest for this environment from the
          # append-only deployment record (§8) -- never from operator input.
          # The script itself rejects a target that is revoked, marked
          # data-incompatible with current schema state, or not the immediately
          # preceding deployment (skipping further back needs a reviewed exception).
          target="$(./scripts/resolve-rollback-target.sh production)"

          if [[ -n "$CONFIRM_DIGEST" ]]; then
            [[ "$CONFIRM_DIGEST" =~ ^sha256:[0-9a-f]{64}$ ]]
            if [[ "$CONFIRM_DIGEST" != "$target" ]]; then
              echo "::error::confirm_digest ($CONFIRM_DIGEST) does not match the resolved rollback target ($target)"
              exit 1
            fi
          fi

          echo "digest=$target" >> "$GITHUB_OUTPUT"

      - name: Assume production deploy role
        uses: aws-actions/configure-aws-credentials@61815dcd50bd041e203e49132bacad1fd04d2708 # v5
        with:
          role-to-assume: arn:aws:iam::111111111111:role/orders-production-deploy
          aws-region: us-east-1

      - name: Verify approved builder produced this artifact
        env:
          GH_TOKEN: ${{ github.token }}
          DIGEST: ${{ steps.resolve.outputs.digest }}
        run: |
          set -euo pipefail
          gh attestation verify \
            "oci://ghcr.io/acme/orders@$DIGEST" \
            --repo acme/orders \
            --signer-workflow acme/orders/.github/workflows/build-and-attest.yml@refs/heads/main

      - name: Deploy previous digest
        env:
          DIGEST: ${{ steps.resolve.outputs.digest }}
        run: ./scripts/deploy-production.sh "ghcr.io/acme/orders@$DIGEST"

      - name: Verify recovery
        run: ./scripts/verify-environment.sh production
```

`resolve-rollback-target.sh` owns the eligibility check described in the comment above: it walks the retained deployment record back to the nearest prior entry and refuses to hand back anything revoked, data-incompatible, or non-adjacent. Its durable append-only source exposes records shaped like:

```json
{"record_id":"dep-0194","environment":"production","digest":"sha256:new...","previous_digest":"sha256:good...","revoked":false,"schema_compatible_back_to":"sha256:good...","verified_at":"2026-09-08T12:00:00Z","writer":"orders-deploy-role/run-8912"}
```

Input `current=sha256:new...` selects adjacent `sha256:good...` because it is unrevoked and schema-compatible. Change only `revoked` to `true`, or set the compatibility boundary past that predecessor, and resolution refuses it. Skipping farther back requires a reviewed exception rather than an operator-supplied digest.

`--signer-workflow` is what makes the attestation check mean something. `--repo acme/orders` alone only proves *some* workflow in that repository produced the attestation — including a test, maintenance, or otherwise-compromised workflow with just enough permissions to sign, which still satisfies repo scope. Pinning `--signer-workflow acme/orders/.github/workflows/build-and-attest.yml@refs/heads/main` narrows trust from "came from this repository" to "came from this specific, reviewed build pipeline" ([gh attestation verify](https://cli.github.com/manual/gh_attestation_verify), checked 2026-08-14).

`configure-aws-credentials` is the **OIDC** (OpenID Connect — a federated identity protocol layered on OAuth 2.0) exchange this job depends on: GitHub issues the running job a short-lived token asserting its exact repository, workflow, and environment identity, and the pinned action trades that token for temporary AWS credentials scoped to the `orders-production-deploy` role. Nothing long-lived is stored in GitHub, and the trust policy on that IAM role — not this workflow file — is what actually restricts which workflows may assume it.

The cloud security owner inspects the live role with `aws iam get-role`; its condition binds `aud=sts.amazonaws.com` and `sub=repo:acme/orders:environment:production`. The approved repository/workflow/ref/environment succeeds; changing a bound claim returns `AccessDenied`. Store the reviewed policy digest and alert when the live query differs, because workflow YAML cannot prove cloud trust.

```json
{"StringEquals":{"token.actions.githubusercontent.com:aud":"sts.amazonaws.com","token.actions.githubusercontent.com:sub":"repo:acme/orders:environment:production"}}
```

Recovery workflows deserve the same identity, environment, and approval controls as normal releases; rollback is still a production deployment.

---

## 8. What You Retain Per Deployment Is What Rollback Can Later Prove

For each deployment, retain:

- new and previous digest;
- task definition, manifest, or chart version;
- [configuration and feature-flag snapshot](05_configuration_versioning_and_recovery.md);
- database migration version and compatibility status;
- routing weights;
- verification output;
- responsible workflow and actor.

This is the append-only deployment record `resolve-rollback-target.sh` (§7) reads. Without a retained compatibility status per digest, there is no way for an automated rollback to tell "binary compatible" apart from "data incompatible" without a human re-deriving it under incident pressure — the retention list above is what makes the resolution step in §7 possible at all.

Test rollback in non-production and during **game days** — planned recovery exercises that inject a controlled failure and measure the actual outcome, rather than a tabletop discussion of what should happen. Measure the actual recovery time, including detection and decision delay.

---

## 9. Measure the System, Not the People Running It

Current guidance from **DORA** (DevOps Research and Assessment — the research program behind the State of DevOps reports and the standard delivery-performance measures) describes five delivery-performance measures. One of them, **deployment rework rate**, is the share of deployments that need a subsequent hotfix, patch, or rollback shortly after release — a proxy for how often a deployment that looked done wasn't.

Current DORA guidance also places failed-deployment recovery time under software-delivery throughput rather than a separate recovery category ([DORA metrics guide](https://dora.dev/guides/dora-metrics/), checked 2026-08-14):

| Dimension | Measure |
|-----------|---------|
| Throughput | Change lead time |
| Throughput | Deployment frequency |
| Throughput | Failed deployment recovery time |
| Instability | Change fail rate |
| Instability | Deployment rework rate |

Attach any of these to a person's or a team's name and two things go wrong. First, whoever is measured starts optimizing the number instead of the outcome it stands in for: deployment frequency turns into splitting one change into more trivial commits, change fail rate turns into redefining what counts as a failure, and recovery time turns into closing the incident before the underlying cause is actually fixed — none of which ships faster or safer software, it just stops the metric from tracking reality. Second, the same number means something different depending on service context: a low-traffic internal batch service and a service absorbing constant customer and third-party traffic will show different natural deployment frequencies and change fail rates even when both teams are equally disciplined. Comparing raw numbers across people or across teams with different blast radii, on-call load, and dependency surfaces treats unlike services as interchangeable and rewards whoever happens to own the easier one.

> **Rule**: Use them to improve the system, not rank individuals.

Start with the three starred system measures; add the conditional ones when they answer a real bottleneck. Also track:

- **★ PR first-feedback and merge time;**
- **★ queue and critical-path duration;**
- **★ flaky-test rate;**
- deployment approval wait;
- rollback test success;
- percentage of deployments with provenance and known previous digest;
- preview-environment leakage and CI cost.

Example: the 30-day p95 queue duration rises from 2 to 11 minutes while execution stays flat at 6 minutes. The team adds runner capacity rather than rewriting tests; the following week's queue p95 returns below 3 minutes. The measure leads to a system change, not a team ranking.

Smaller batch size often improves both throughput and recovery.

---

## 10. An Incident Should Change the System, Not Just End It

```text
deployment-related incident
    ↓ contain and recover
    ↓ identify missing or misleading evidence
    ↓ add test, policy, signal, or safer rollout step
    ↓ verify the control in a game day
    ↓ remove temporary incident-only workaround
```

Not every incident requires another gate. Sometimes the right fix is clearer ownership, a smaller change, better rollback compatibility, or removing a fragile mechanism.

---

## 11. The Same Few Failures Keep Recurring

**⚠️ Health checks are the only production test**

The process is alive while the business path is broken. Add synthetic and domain signals.

**⚠️ Rollback uses an old Git commit and rebuilds it**

Dependencies and base images have moved. Redeploy the retained immutable artifact.

**⚠️ Automatic rollback loops**

The platform repeatedly moves between two unhealthy versions. Bound attempts and enter an incident state.

> **Edge case:** automatic rollback loops only show up once automated rollback triggers are enabled without a bound on attempts — read this before you wire up that trigger, not before.

**⚠️ The release caused data corruption**

Compute rollback does not repair data. Stop writes if necessary, reconcile affected records, and roll forward with a corrective migration.

> **Key insight**: Deployment success is an evidence chain from platform convergence to user-visible behavior, not the exit status of the deployment command.

---

## 12. Primary Sources for the Claims Above

- [Deploying with GitHub Actions](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments)
- [Amazon ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html)
- [gh attestation verify](https://cli.github.com/manual/gh_attestation_verify) (checked 2026-08-14)
- [DORA software delivery performance metrics](https://dora.dev/guides/dora-metrics/) (checked 2026-08-14)

---

**Next**: [Configuration Versioning and Recovery](05_configuration_versioning_and_recovery.md)
