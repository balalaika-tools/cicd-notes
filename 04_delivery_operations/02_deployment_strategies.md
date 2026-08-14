# Deployment Strategies and Progressive Delivery

> **Who this is for**: Engineers choosing how a new version receives production traffic. Read [Environments and Artifact Promotion](01_environments_and_promotions.md) first.

## The short version

Deploy a bad build to every instance at once and every user hits it before anyone notices — there's no window between "code is running" and "everyone is exposed." **Progressive delivery** is the fix: deliberately increase exposure — the fraction of instances or requests routed to the new version — in small steps, and widen exposure only while each step's evidence stays healthy.

**What you need (4 things):**

1. A named candidate version — a container image digest, the immutable content hash that pins exactly which build is running, not a mutable tag.
2. A traffic mechanism that can move a known, small slice of instances or requests onto that candidate.
3. Version-segmented health signals: error rate and latency measured separately for candidate vs. baseline, never blended together.
4. A fixed decision rule: the exact threshold that means "continue," and the exact threshold that means "stop and roll back."

**Worked example:**

```text
orders service — candidate sha256:4f9c1a...

traffic:  0% ──5%──▶ 25% ──▶ 50% ──▶ 100%
                 │
                 └── bake: fixed observation window held after every step

step at 5% weight, after the bake window:
  requests observed:   1,204
  error rate:           0.25%   (ceiling 1%)
  latency:              238ms   (ceiling 400ms)
  decision:             CONTINUE → widen to 25%

same step, if telemetry is missing instead:
  requests observed:   0
  decision:             STOP → hold at 0% (no data is not a healthy signal)
```

**Success signal:** every metric inside its threshold for the whole bake window means continue to the next step; any threshold breached, or zero candidate requests observed, means stop and return traffic to 0%.

**Not handled yet:** whether the previous version can still read what the candidate already wrote ([state compatibility](#9-rollback-only-works-if-the-previous-version-can-still-read-the-data)), how a traffic percentage is actually enforced at the network layer ([routing](#5-canary-limits-exposure-by-watching-metrics-before-widening-traffic)), how long a flag stays in the codebase after rollout finishes ([flag lifecycle](#6-feature-flags-separate-exposure-from-deployment-not-from-compatibility)), and what a failed rollout needs to restore ([recovery](04_verification_observability_and_rollback.md)).

---

## 1. Deployment Is Not Release

A team ships a new binary to every production pod in one rolling restart. Two minutes later, every user in every region is served by the new code — at no point in that rollout did anyone choose who saw the new behavior, only whether the process was up. If the build has a bug, it isn't a few users' problem; it's everyone's, simultaneously, and the only lever left is a fleet-wide emergency rollback.

That happens because two different actions get treated as one:

```text
deployment
    make code available in production infrastructure

release
    expose behavior to users
```

Conflating them removes a lever. Deploying without releasing lets you get new code running — and verify it's operationally sound — before a single user's request depends on it.

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

## 2. Each Strategy Buys Safety by Spending Capacity, Complexity, or Time

Before comparing rows, know what each name refers to:

- **Recreate** — stop every old instance, then start every new one. Simplest possible sequence; there's a window where nothing serves traffic at all.
- **Rolling** — replace old instances with new ones a few at a time, so old and new versions serve real traffic side by side during the transition.
- **Blue-green** — bring the new version up fully in a separate, idle environment, then switch all traffic to it in one move, keeping the old environment live as a fast switchback.
- **Canary** — send a small, growing percentage of real traffic to the new version while the rest keeps hitting the old one, watching metrics before widening exposure further.
- **Shadow** — send a copy of real requests to the new version without using its response, observing behavior with no effect on what the user sees.

| Strategy | Extra capacity | Traffic control | Rollback speed | Main constraint |
|----------|----------------|-----------------|----------------|-----------------|
| Recreate | Low | None | Slow/downtime | Service unavailable during replacement |
| Rolling | Moderate | Instance replacement | Moderate | Old and new coexist |
| Blue-green | High | One major switch | Fast switchback | Duplicate environment cost and state |
| Canary | Moderate | Percentage/cohort | Fast for limited exposure | Requires reliable metrics and routing |
| Feature flag | Application-dependent | User/behavior exposure | Very fast flag change | Two code paths and flag debt |
| Shadow | High for duplicated processing | No user response from candidate | No user rollback needed | Side effects and data privacy |

> **Core:** the safest choice depends on state compatibility, observability, traffic control, capacity, and consequence for the specific change being shipped — not on which strategy is most familiar or currently fashionable.

---

## 3. Rolling Deployments Avoid Downtime by Running Old and New Together

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

> **Production:** the controls below are what keep a rolling deployment from turning a bad build into an outage — required before this ships, not needed to understand the shape above.

Control:

- maximum unavailable capacity;
- maximum surge;
- readiness and startup probes;
- connection draining;
- deployment timeout;
- platform circuit breaker;
- backward-compatible schema and protocol.

AWS **ECS** (Elastic Container Service, AWS's managed container orchestrator) rolling deployments can use a **deployment circuit breaker** — a failure detector that watches task health during the rollout, stops the rollout once a failure threshold is crossed, and can automatically restore the last completed deployment — together with CloudWatch alarms that feed it failure signals.

---

## 4. Blue-Green Buys Instant Rollback by Duplicating Everything

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

> **Production:** these are the specific ways a "flip traffic back" plan quietly stops being available by the time you need it.

Watch for:

- data writes that make switching back unsafe;
- background workers running in both environments;
- session or cache incompatibility;
- DNS propagation if DNS is used as the switch;
- cleanup happening before the rollback window closes.

Blue-green changes compute exposure; it does not duplicate the database automatically.

---

## 5. Canary Limits Exposure by Watching Metrics Before Widening Traffic

```text
candidate traffic:
0% → 5% → 25% → 50% → 100%
     │     │      │
     └─────┴──────┴── bake + evaluate at each step
```

Each arrow holds for a **bake** — the fixed observation window described above, held after every traffic step — before the controller evaluates and either widens exposure or stops.

Choose canary signals before rollout:

- request error rate by version;
- latency percentiles by version;
- saturation and restarts;
- dependency errors;
- business conversions or rejected operations;
- invariant or data-quality violations.

> **Production:** a canary is only as good as the controller enforcing it — the interface below is what turns the diagram above into an actual gate.

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

The flags encode the decision rule directly: a max error rate, and a max **p95** latency — the value under which 95% of measured requests fall, so a handful of slow outliers don't block promotion the way the true worst case would.

One evaluated window looks like this:

```text
$ ./scripts/verify-canary-window.sh --service orders --candidate-digest sha256:4f9c1a... \
    --minutes 10 --max-error-rate 0.01 --max-p95-latency-ms 400

candidate requests observed:  1204
candidate error rate:         0.0025   (threshold 0.01)
candidate p95 latency:        238ms    (threshold 400ms)
decision: PROMOTE  (5% -> 25%)
```

And the failure mode that isn't a threshold breach at all:

```text
candidate requests observed:  0
decision: HALT  (no candidate telemetry in the window — missing data is treated as failure, not as a clean signal)
```

Those scripts must query version-segmented metrics, handle low traffic, and stop or roll back when data is missing. "No metrics" is not success.

AWS ECS canary deployments support a small initial traffic shift followed by full traffic, bake time, lifecycle hooks, and alarm-driven rollback. Other platforms and progressive-delivery controllers support more steps and cohort strategies.

---

## 6. Feature Flags Separate Exposure From Deployment, Not From Compatibility

Classify flags:

| Flag | Lifetime | Example |
|------|----------|---------|
| Release flag | Days or weeks | Gradually expose a new checkout |
| Operational kill switch | Long-lived, tested | Disable expensive recommendation calls |
| Experiment | Bounded by experiment | Compare ranking algorithms |
| Permission flag | Long-lived policy | Enable a tenant capability |

> **Production:** the rules below are what keep a flag from becoming its own outage — an untested combination or stale cached state is exactly as dangerous as a bad deploy.

Operational rules:

- safe default when the flag service is unavailable;
- audit changes and restrict production write access;
- include flag state in deployment and incident context;
- test important on/off combinations;
- cache with bounded staleness;
- remove release flags after full adoption.

A flag controls which code path runs; it has no say over what shape the data underneath is in. Say a migration changes a column's format, and either an old instance that hasn't redeployed yet, or any request routed to a cohort where the flag is still off, reads a row the new code already wrote. The flag never touches that instance's code — it still runs the old parser — and the row is now in a shape that parser cannot read, so the request fails or misreads data regardless of what the flag is set to. Hiding an incompatible schema migration behind a feature flag doesn't make the migration compatible; it just guarantees that some cohort hits the incompatible shape while believing the flag protected them.

---

## 7. Shadow Traffic Tests Candidates With Zero User-Facing Risk

> **Edge case:** reach for shadowing only when you need production-realistic input with zero chance of affecting a real user. Most changes are better served by canary, which does expose users, but under a controlled and reversible slice.

Shadowing sends a copy of production requests to a candidate but ignores its response:

```text
user request ──> current version ──> user response
       │
       └────copy────> candidate ──> metrics only
```

Use it for performance, compatibility, or model comparison. Strip sensitive data when required and suppress side effects such as emails, payments, or writes.

Shadow success does not prove the candidate's response is acceptable to users unless outputs are compared with domain-aware tolerances.

---

## 8. Match the Strategy to the Change's Risk, Not to Habit

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

**Expand-contract** is a compatibility-preserving sequence for schema change: add the new shape while the old one keeps working, migrate reads and writes over to it, and only remove the old shape once nothing still depends on it — never drop or rewrite a column in place while old and new code might both be running.

Each extra mechanism adds operational complexity. Use the smallest combination that controls the actual failure modes.

---

## 9. Rollback Only Works if the Previous Version Can Still Read the Data

> **Core:** rollback is the default response to a bad deploy, but only while the previous version is still data-compatible with what's already been written.

Rollback is appropriate when:

- the previous artifact remains data-compatible;
- the failure is in application code or configuration;
- rollback is faster and lower risk than a fix.

> **Edge case:** these are the situations where rollback stops being an option and roll-forward becomes the only safe move.

Roll forward when:

- the new version has already written incompatible data;
- an infrastructure or schema change cannot safely reverse;
- the failure is understood and a narrow fix is ready;
- the old version has a known critical vulnerability.

> **Production:** none of the above is decidable in the middle of an incident unless this state was already captured before the incident started.

Always retain:

- previous artifact digest and task definition;
- release manifest and [configuration version](05_configuration_versioning_and_recovery.md);
- traffic-routing state;
- flag state;
- migration compatibility window.

> **Key insight**: rollout strategy — rolling, blue-green, canary, shadow — only ever limits how many compute instances or users are exposed to a bad version. Whether rollback is actually safe is decided somewhere else entirely: by whether data and protocol compatibility still hold between the previous version and whatever the new one already wrote.

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
