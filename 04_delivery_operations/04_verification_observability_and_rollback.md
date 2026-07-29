# Verification, Observability, and Rollback

> **Who this is for**: Teams defining when a deployment is actually successful and how to recover when it is not. Read [Infrastructure and Database Changes](03_infrastructure_and_database_changes.md) first.

---

## 1. Deployment Command Success Is Not Service Success

Verify in layers:

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

---

## 2. Design Useful Health Endpoints

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

## 3. Run Bounded Smoke Tests

```bash
#!/usr/bin/env bash
set -euo pipefail

base_url="${1:?usage: smoke.sh BASE_URL EXPECTED_COMMIT}"
expected_commit="${2:?usage: smoke.sh BASE_URL EXPECTED_COMMIT}"

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
test "$actual_commit" = "$expected_commit"

curl \
  --connect-timeout 5 \
  --max-time 20 \
  --fail \
  --silent \
  --show-error \
  "$base_url/api/v1/catalog?limit=1" > /dev/null
```

Production smoke tests must:

- avoid destructive or duplicate side effects;
- use dedicated synthetic identities and data;
- have timeouts;
- verify the expected release;
- produce diagnostic evidence without exposing secrets.

---

## 4. Attach Deployment Markers to Telemetry

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

Add release identity to logs, traces, metrics, and error reports. During an incident, responders should be able to move from a latency spike to the deployment and then to its source and artifact.

Avoid high-cardinality metric labels for commit or digest in every series. Use deployment events, logs, trace resource attributes, or a bounded release label appropriate to the telemetry backend.

---

## 5. Gate on SLO-Aware Signals

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

Missing telemetry should stop a risky progressive rollout. An observability outage removes evidence; it does not prove the release healthy.

---

## 6. Make Recovery an Explicit State Machine

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

## 7. Implement Digest-Based Rollback

```yaml
name: Roll Back Production

on:
  workflow_dispatch:
    inputs:
      previous_digest:
        required: true
        type: string
      incident:
        required: true
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
      - name: Validate recovery request
        env:
          DIGEST: ${{ inputs.previous_digest }}
          INCIDENT: ${{ inputs.incident }}
        run: |
          set -euo pipefail
          [[ "$DIGEST" =~ ^sha256:[0-9a-f]{64}$ ]]
          [[ "$INCIDENT" =~ ^INC-[0-9]+$ ]]

      - name: Verify previous artifact
        env:
          GH_TOKEN: ${{ github.token }}
          DIGEST: ${{ inputs.previous_digest }}
        run: |
          gh attestation verify \
            "oci://ghcr.io/acme/orders@$DIGEST" \
            --repo acme/orders

      - name: Deploy previous digest
        env:
          DIGEST: ${{ inputs.previous_digest }}
        run: ./scripts/deploy-production.sh "ghcr.io/acme/orders@$DIGEST"

      - name: Verify recovery
        run: ./scripts/verify-environment.sh production
```

The example assumes authentication is added through a pinned cloud OIDC action before deployment. Recovery workflows deserve the same identity and environment controls as normal releases.

---

## 8. Retain Recovery Inputs

For each deployment, retain:

- new and previous digest;
- task definition, manifest, or chart version;
- configuration and feature-flag snapshot;
- database migration version and compatibility status;
- routing weights;
- verification output;
- responsible workflow and actor.

Test rollback in non-production and during game days. Measure the actual recovery time, including detection and decision delay.

---

## 9. Measure the Delivery System

Current DORA guidance describes five delivery-performance measures:

| Dimension | Measure |
|-----------|---------|
| Throughput | Change lead time |
| Throughput | Deployment frequency |
| Instability | Change fail rate |
| Instability | Deployment rework rate |
| Recovery | Failed deployment recovery time |

Use them to improve the system, not rank individuals.

Also track:

- PR first-feedback and merge time;
- queue and critical-path duration;
- flaky-test rate;
- deployment approval wait;
- rollback test success;
- percentage of deployments with provenance and known previous digest;
- preview-environment leakage and CI cost.

Smaller batch size often improves both throughput and recovery.

---

## 10. Incident Feedback Loop

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

## 11. Common Failure Modes

**Health checks are the only production test**

The process is alive while the business path is broken. Add synthetic and domain signals.

**Rollback uses an old Git commit and rebuilds it**

Dependencies and base images have moved. Redeploy the retained immutable artifact.

**Automatic rollback loops**

The platform repeatedly moves between two unhealthy versions. Bound attempts and enter an incident state.

**The release caused data corruption**

Compute rollback does not repair data. Stop writes if necessary, reconcile affected records, and roll forward with a corrective migration.

---

## 12. References

- [Deploying with GitHub Actions](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments)
- [Amazon ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html)
- [DORA software delivery performance metrics](https://dora.dev/guides/dora-metrics/)

---

**Next**: [End-to-End Production Example](../05_reference/01_end_to_end_production_example.md)
