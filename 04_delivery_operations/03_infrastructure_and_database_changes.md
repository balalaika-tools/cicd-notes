# Infrastructure and Database Changes

> **Who this is for**: Teams delivering Terraform and schema changes alongside applications. Read [Deployment Strategies and Progressive Delivery](02_deployment_strategies.md) first.

---

## 1. Separate Pipelines by Recovery Boundary

```text
application deployment
    usually redeploy previous immutable artifact

infrastructure change
    may replace or destroy stateful resources

database migration
    may transform persistent data irreversibly
```

Coordinate these flows, but give each:

- its own permission scope;
- concurrency control;
- review ownership;
- verification;
- recovery plan;
- deployment record.

One broad `deploy.sh` obscures which operation failed and encourages all-powerful credentials.

---

## 2. Terraform Pull-Request Flow

```text
pull request
├── terraform fmt -check
├── terraform init -backend=false   where possible
├── terraform validate
├── provider and module lock validation
├── security and policy checks
└── speculative plan with controlled read credentials
      └── render a sanitized summary for review
```

Example:

```yaml
jobs:
  terraform-plan:
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    runs-on: ubuntu-latest
    timeout-minutes: 30
    concurrency:
      group: tf-plan-production-${{ github.event.pull_request.number }}
      cancel-in-progress: true
    defaults:
      run:
        working-directory: infra/environments/production
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4

      - name: Format and initialize
        run: |
          set -euo pipefail
          terraform fmt -check -recursive
          terraform init -input=false

      - name: Validate and plan
        run: |
          set -euo pipefail
          terraform validate
          terraform plan \
            -input=false \
            -lock-timeout=5m \
            -no-color > plan.txt
```

Do not give untrusted fork code production read credentials. Run only static validation for forks, or use a separately reviewed mechanism for speculative plans.

Sanitize plan output before posting it to a pull request. Terraform plans can include sensitive values even when terminal output attempts to redact some fields.

---

## 3. Apply the Exact Approved Plan or Re-Approve

There are two sound models:

### Saved plan

```text
trusted plan job
    ↓ save plan + complete working directory securely
    ↓ approval identifies that exact plan
    ↓ apply saved plan
```

A saved Terraform plan includes configuration and prior state data and may contain sensitive values. Protect it as a privileged artifact. The apply environment must use compatible paths, operating system, architecture, Terraform version, and provider binaries.

### New plan after merge

```text
PR speculative plan for review
    ↓ merge exact commit
    ↓ create final plan against current state
    ↓ review/automated policy gate
    ↓ apply that final plan
```

Do not silently apply a different final plan from the one a human reviewed. If drift or another apply changes the result, invalidate or re-approve the plan.

---

## 4. Serialize State Changes

Use both:

- a Terraform backend that supports state locking;
- GitHub or platform concurrency per state/workspace.

```yaml
concurrency:
  group: terraform-production-network
  cancel-in-progress: false
```

Never disable state locking to “fix” a busy pipeline. `terraform force-unlock` should target only a known stale lock from your own failed operation after confirming no writer is active.

Keep distinct state boundaries small enough to reduce blast radius but not so fragmented that every change requires complex cross-state orchestration.

---

## 5. Detect and Reconcile Drift

Scheduled drift detection:

```text
scheduled read-only plan
    ├── no change → record healthy
    └── drift
         ├── expected emergency/manual change → import or codify through PR
         └── unexpected change → investigate and reconcile
```

Do not automatically overwrite unexplained production drift. It may indicate an incident, emergency repair, or compromised identity.

Restrict direct console changes. When emergency manual changes occur, create the corresponding code change promptly.

---

## 6. Run Database Migrations Once

❌ Every application replica runs migrations at startup.

✅ A dedicated, serialized migration job runs with explicit identity and records the schema version.

```yaml
jobs:
  migrate:
    environment: production
    concurrency:
      group: database-production-orders
      cancel-in-progress: false
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - name: Run the target migration
        env:
          DATABASE_URL: ${{ secrets.MIGRATION_DATABASE_URL }}
          TARGET_REVISION: "2026_07_29_01"
        run: |
          set -euo pipefail
          python -m alembic upgrade "$TARGET_REVISION"
          python -m alembic current
```

Prefer short-lived database identity or a secret broker when the platform supports it. The migration role should have schema permissions that the application runtime role does not.

---

## 7. Use Expand and Contract

Example: replace `customer_name` with structured names.

```text
Release N: expand
    add nullable given_name and family_name
    old code still works

Release N+1: compatible writers
    write old and new fields
    observe mismatches

Backfill
    bounded, resumable batches
    verify counts and checksums

Release N+2: switch reads
    read new fields
    retain fallback temporarily

Release N+3: contract
    stop writing customer_name
    remove old field after compatibility window
```

Every intermediate state must support the versions that can coexist during rolling deployment and rollback.

---

## 8. Design Backfills as Production Workloads

A safe backfill is:

- idempotent and resumable;
- partitioned into bounded batches;
- rate-limited;
- observable;
- pauseable;
- protected against concurrent execution;
- verified with domain invariants;
- isolated from request-path resource exhaustion.

```bash
python -m tools.backfill_customer_names \
  --after-id "$CHECKPOINT" \
  --batch-size 500 \
  --max-rows-per-second 200 \
  --checkpoint-table operations.backfill_progress
```

Do not hide a multi-hour backfill inside a deployment job timeout. Treat it as an operation with its own lifecycle and release dependency.

---

## 9. Backups Are Not Rollbacks

Before a high-risk migration:

- verify backup recency and scope;
- test restore procedures;
- measure restore time against recovery objectives;
- know which later writes would be lost;
- capture schema and migration version;
- define who authorizes restore.

Restoring a database can be much more disruptive than rolling forward with a corrective migration. Prefer compatibility and reversible steps over relying on restore.

---

## 10. Coordinate Release Order

Typical safe sequence:

```text
1. additive infrastructure
2. expand schema
3. compatible application version
4. controlled backfill
5. switch reads or traffic
6. observe through compatibility window
7. remove old application path
8. contract schema/infrastructure later
```

Use feature flags for behavior, not as a substitute for data compatibility.

---

## 11. Common Failure Modes

**A PR plan is applied days later**

State and configuration may have changed. Create and approve a fresh final plan or use a system that invalidates stale saved plans.

**Infrastructure and application share one admin role**

A compromised application deployment can modify networking, IAM, and data services. Split roles by operation.

**A migration succeeds but locks the table too long**

Test on representative scale, use online techniques, set lock/statement timeouts, and monitor database saturation.

**Rollback deploys an old binary after schema contraction**

Keep the compatibility window until old versions are no longer a valid recovery target.

---

## 12. References

- [Running Terraform in automation](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform)
- [`terraform plan`](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [`terraform apply`](https://developer.hashicorp.com/terraform/cli/commands/apply)
- [Terraform state locking](https://developer.hashicorp.com/terraform/language/state/locking)

---

**Next**: [Verification, Observability, and Rollback](04_verification_observability_and_rollback.md)
