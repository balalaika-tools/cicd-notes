# Deployment Strategies and Progressive Delivery

> **Who this is for**: Engineers choosing how a new version receives production traffic. Read [Environments and Artifact Promotion](01_environments_and_promotions.md) first.

---

## 1. Deployment Is Not Release

```text
deployment
    make code available in production infrastructure

release
    expose behavior to users
```

Feature flags allow the two to happen separately:

```text
deploy hidden code
    ↓ verify operational compatibility
    ↓ enable for employees
    ↓ enable for 5% of users
    ↓ expand by cohort
    ↓ remove old path and stale flag later
```

Flags reduce exposure risk but add state and testing combinations. Every flag needs an owner, purpose, default, and removal date.

---

## 2. Strategy Comparison

| Strategy | Extra capacity | Traffic control | Rollback speed | Main constraint |
|----------|----------------|-----------------|----------------|-----------------|
| Recreate | Low | None | Slow/downtime | Service unavailable during replacement |
| Rolling | Moderate | Instance replacement | Moderate | Old and new coexist |
| Blue-green | High | One major switch | Fast switchback | Duplicate environment cost and state |
| Canary | Moderate | Percentage/cohort | Fast for limited exposure | Requires reliable metrics and routing |
| Feature flag | Application-dependent | User/behavior exposure | Very fast flag change | Two code paths and flag debt |
| Shadow | High for duplicated processing | No user response from candidate | No user rollback needed | Side effects and data privacy |

The safest choice depends on state compatibility, observability, traffic control, capacity, and consequence—not fashion.

---

## 3. Rolling Deployment

```text
time ─────────────────────────────────────────────>

old old old old
new old old old
new new old old
new new new old
new new new new
```

Use when:

- instances are stateless or tolerate replacement;
- old and new versions can serve simultaneously;
- capacity supports surge or temporary reduction;
- readiness checks accurately represent service ability.

Control:

- maximum unavailable capacity;
- maximum surge;
- readiness and startup probes;
- connection draining;
- deployment timeout;
- platform circuit breaker;
- backward-compatible schema and protocol.

AWS ECS rolling deployments can use a deployment circuit breaker and CloudWatch alarms to detect failure and automatically roll back to the last completed deployment.

---

## 4. Blue-Green Deployment

```text
                 ┌──────────────┐
traffic ────────>│ blue/current │
                 └──────────────┘

deploy + test     ┌──────────────┐
without users ──>│ green/new    │
                 └──────────────┘

switch traffic:
traffic ────────> green/new
retain blue for bounded rollback window
```

Use when a traffic switch is available and fast rollback justifies duplicate capacity.

Watch for:

- data writes that make switching back unsafe;
- background workers running in both environments;
- session or cache incompatibility;
- DNS propagation if DNS is used as the switch;
- cleanup happening before the rollback window closes.

Blue-green changes compute exposure; it does not duplicate the database automatically.

---

## 5. Canary Deployment

```text
candidate traffic:
0% → 5% → 25% → 50% → 100%
     │     │      │
     └─────┴──────┴── bake + evaluate at each step
```

Choose canary signals before rollout:

- request error rate by version;
- latency percentiles by version;
- saturation and restarts;
- dependency errors;
- business conversions or rejected operations;
- invariant or data-quality violations.

Concrete controller interface:

```bash
set -euo pipefail

for weight in 5 25 50 100; do
  ./scripts/set-canary-weight.sh \
    --service orders \
    --digest "$IMAGE_DIGEST" \
    --percent "$weight"

  ./scripts/verify-canary-window.sh \
    --service orders \
    --candidate-digest "$IMAGE_DIGEST" \
    --minutes 10 \
    --max-error-rate 0.01 \
    --max-p95-latency-ms 400
done
```

Those scripts must query version-segmented metrics, handle low traffic, and stop or roll back when data is missing. “No metrics” is not success.

AWS ECS canary deployments support a small initial traffic shift followed by full traffic, bake time, lifecycle hooks, and alarm-driven rollback. Other platforms and progressive-delivery controllers support more steps and cohort strategies.

---

## 6. Feature Flags

Classify flags:

| Flag | Lifetime | Example |
|------|----------|---------|
| Release flag | Days or weeks | Gradually expose a new checkout |
| Operational kill switch | Long-lived, tested | Disable expensive recommendation calls |
| Experiment | Bounded by experiment | Compare ranking algorithms |
| Permission flag | Long-lived policy | Enable a tenant capability |

Operational rules:

- safe default when the flag service is unavailable;
- audit changes and restrict production write access;
- include flag state in deployment and incident context;
- test important on/off combinations;
- cache with bounded staleness;
- remove release flags after full adoption.

Do not use a feature flag to hide an incompatible schema migration.

---

## 7. Shadow and Mirrored Traffic

Shadowing sends a copy of production requests to a candidate but ignores its response:

```text
user request ──> current version ──> user response
       │
       └────copy────> candidate ──> metrics only
```

Use it for performance, compatibility, or model comparison. Strip sensitive data when required and suppress side effects such as emails, payments, or writes.

Shadow success does not prove the candidate's response is acceptable to users unless outputs are compared with domain-aware tolerances.

---

## 8. Match Strategy to Change Risk

| Change | Default strategy |
|--------|------------------|
| Stateless patch with strong health checks | Rolling |
| High-risk API change with compatible data | Canary |
| Runtime/platform upgrade needing quick switchback | Blue-green |
| User-facing behavior separable from deploy | Feature flag plus canary |
| New model or query engine | Shadow, then canary |
| Destructive data migration | Staged migration; rollout strategy alone is insufficient |

Combine techniques:

```text
blue-green infrastructure
    + canary traffic
    + feature-flagged user exposure
    + expand-contract data migration
```

Each extra mechanism adds operational complexity. Use the smallest combination that controls the actual failure modes.

---

## 9. Rollback and Roll-Forward

Rollback is appropriate when:

- the previous artifact remains data-compatible;
- the failure is in application code or configuration;
- rollback is faster and lower risk than a fix.

Roll forward when:

- the new version has already written incompatible data;
- an infrastructure or schema change cannot safely reverse;
- the failure is understood and a narrow fix is ready;
- the old version has a known critical vulnerability.

Always retain:

- previous artifact digest and task definition;
- release manifest and configuration version;
- traffic-routing state;
- flag state;
- migration compatibility window.

---

## 10. Common Failure Modes

**Readiness returns success before dependencies work**

New instances receive traffic and fail. Separate startup, liveness, and readiness semantics and test them under failure.

**Canary metrics mix old and new versions**

The aggregate looks healthy while the candidate fails. Label telemetry by release/digest and compare against a baseline.

**Rollback removes compute but not side effects**

Messages, emails, and data writes remain. Make side effects idempotent and include reconciliation in recovery.

**Blue is deleted immediately**

The switchback mechanism disappears before the bake window completes.

---

## 11. References

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Amazon ECS deployment failure detection](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-failure-detection.html)
- [Amazon ECS canary deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/canary-deployment.html)

---

**Next**: [Infrastructure and Database Changes](03_infrastructure_and_database_changes.md)
