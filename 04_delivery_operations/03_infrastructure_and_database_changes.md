# Infrastructure and Database Changes

> **Who this is for**: Teams delivering Terraform and schema changes alongside applications. Read [Deployment Strategies and Progressive Delivery](02_deployment_strategies.md) first.

## The short version

A rollback that redeploys the previous container image cannot undo a destroyed subnet or a dropped database column — application, infrastructure, and database changes have different **recovery boundaries**, the point past which an operation can no longer simply be reversed. Route each through its own pipeline stage with its own credentials, so a Terraform plan gets applied exactly as reviewed and a migration runs once, against a named target, with a recorded result.

**What you need (3 things):**

1. A Terraform plan for one bounded change, produced with read-only plan credentials.
2. An approval (human or policy) that references that exact plan, not one re-derived later.
3. A single serialized apply step that leaves behind a record of what actually changed.

**The code:**

```yaml
- name: Plan
  run: |
    set -euo pipefail
    terraform init -input=false
    terraform plan -input=false -lock-timeout=5m -out=tfplan
    terraform show -no-color tfplan > plan.txt
    # the line that matters for review, e.g.:
    # Plan: 1 to add, 1 to change, 0 to destroy.

- name: Apply the reviewed plan
  run: |
    set -euo pipefail
    terraform apply -input=false -lock-timeout=5m tfplan | tee apply.log
    terraform show -json > post-apply-state.json
    terraform output -json > post-apply-outputs.json
```

Keep `tfplan`/its JSON as approval evidence; it describes the saved plan and prior/planned values, not resulting applied state. The recovery record links the plan digest, `apply.log`, post-apply state/outputs or provider observation, workflow run, and apply identity.

**Success signal:** `terraform plan` exits `0` and prints a `Plan: N to add, M to change, P to destroy` summary matching the intended change (here, `1 to add, 1 to change, 0 to destroy`); `terraform apply` then exits `0` against that same saved plan, with no `Error: Saved plan is stale` message.

**Not handled yet:** [custody of a saved plan between review and apply](#3-apply-the-exact-approved-plan-or-re-approve), [state locking under concurrent applies](#4-serialize-state-changes), [drift detection](#5-detect-and-reconcile-drift), [backfills as their own workload](#8-design-backfills-as-production-workloads), and [schema contraction](#7-use-expand-and-contract).

---

## 1. Separate Pipelines by Recovery Boundary

A single `deploy.sh` that runs `terraform apply` and then `alembic upgrade` right after redeploying application containers treats "deploy" as one operation. It isn't. If the new containers misbehave, rolling back to the previous immutable image is cheap — that build still exists and still works. But by the time anyone rolls back, the Terraform apply may already have destroyed a subnet the previous version depends on, and the migration may already have dropped a column the previous code reads. Neither comes back because the container rolled back: the infrastructure and database steps crossed their own recovery boundary, and one script hid that they were different operations with different ways back.

```text
application deployment
    usually redeploy previous immutable artifact

infrastructure change
    may replace or destroy stateful resources

database migration
    may transform persistent data irreversibly
```

Coordinate these flows — a release still needs infrastructure, then schema, then application changes to land in an order that keeps the system running — but give each its own permission scope, concurrency control, review ownership, verification, recovery plan, and deployment record. One broad `deploy.sh` also tends to run under one credential powerful enough to touch all three, which section 11 revisits as a specific failure mode.

> **Core:** everything below follows from one idea — a pipeline's recovery boundary determines its permission scope, its review process, and what "rollback" even means for it. Split pipelines along recovery boundaries, not along team or repository lines.

> **Key insight**: pipeline boundaries should follow recovery boundaries, not team or repository boundaries — application, infrastructure, and data changes cannot be reversed by the same operation, so merging them into one deploy script doesn't save effort, it just hides which recovery path you actually need when something goes wrong.

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

A **speculative plan** is a preview that is never applied: `terraform plan` computed against real infrastructure state, but run with read-only credentials scoped so this job can see the diff and nothing else. That separation is what lets an untrusted pull request see real output without being trusted to change anything.

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

      - name: Assume the production read role
        uses: aws-actions/configure-aws-credentials@61815dcd50bd041e203e49132bacad1fd04d2708 # v5
        with:
          role-to-assume: arn:aws:iam::123456789012:role/terraform-production-plan
          aws-region: eu-west-1

      - name: Assert identity, backend, and workspace
        run: |
          set -euo pipefail
          test "$(aws sts get-caller-identity --query Account --output text)" = 123456789012
          terraform init -input=false -backend-config="key=orders/production.tfstate"
          test "$(terraform workspace show)" = production

      - name: Format and initialize
        run: |
          set -euo pipefail
          terraform fmt -check -recursive
          terraform init -input=false -backend-config="key=orders/production.tfstate"

      - name: Validate and plan
        run: |
          set -euo pipefail
          terraform validate
          terraform plan \
            -input=false \
            -lock-timeout=5m \
            -no-color > plan.txt

      - name: Publish a sanitized immutable review artifact
        env:
          GH_TOKEN: ${{ github.token }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: |
          set -euo pipefail
          ./scripts/sanitize-terraform-plan.sh plan.txt > plan.sanitized.txt
          sha256sum plan.sanitized.txt > plan.sanitized.txt.sha256
          gh pr comment "$PR_NUMBER" --body "Terraform plan artifact: tf-plan-$GITHUB_RUN_ID; digest: $(cut -d' ' -f1 plan.sanitized.txt.sha256)"
      - uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4
        with:
          name: tf-plan-${{ github.run_id }}
          path: |
            infra/environments/production/plan.sanitized.txt
            infra/environments/production/plan.sanitized.txt.sha256
          if-no-files-found: error
```

The payoff a reviewer checks is the `Plan: N to add, M to change, P to destroy` line at the end of `plan.txt` — that summary is what should match the PR's stated intent. Both `terraform init` and `terraform plan` exit `0` on success regardless of which backend or workspace they ran against, so a clean exit status alone proves nothing about *where* the plan ran. The common silent failure is a plan against the wrong backend or workspace: initialization succeeds, the plan prints a small, clean-looking summary, and it's clean only because it's comparing against the wrong state file. Confirm the backend key and `terraform workspace show` in the job output before trusting a suspiciously quiet plan.

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

Do not silently apply a different final plan from the one a human reviewed. If **drift** — a difference between the declared configuration and the live infrastructure state — has appeared since review, or another apply already changed the result, invalidate the stale plan and require a fresh review rather than applying it anyway.

> **Core:** either model works, but both share one rule — the thing that gets applied must be provably the thing that got approved, not a plausible re-derivation of it.

---

## 4. Serialize State Changes

Two pull requests merge minutes apart and both trigger an apply against the same Terraform workspace. Without protection, both applies read the same prior state, compute their plans against it, and apply in whatever order the runners happen to finish — the second apply's write can silently overwrite resource IDs the first apply just recorded, leaving the state file out of sync with what's actually deployed. **State locking** is the backend holding a mutual-exclusion lock during plan and apply so only one write to a given state file can happen at a time; a second run blocks (or fails fast) instead of racing the first.

Use both:

- a Terraform backend that supports state locking;
- GitHub or platform concurrency per state/workspace.

```yaml
concurrency:
  group: terraform-production-network
  cancel-in-progress: false
```

> **Production:** locking only helps if nothing routes around it. Never disable state locking to "fix" a busy pipeline — that reintroduces the exact race it exists to prevent. Keep distinct state boundaries small enough to reduce blast radius but not so fragmented that every change requires complex cross-state orchestration.

> **Edge case:** `terraform force-unlock` should target only a known stale lock left by your own failed operation, and only after confirming no other writer is active — running it against a lock someone else currently holds reintroduces the same concurrent-write race this section exists to prevent.

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

Run the read-only identity and backend/workspace assertions from section 2, then:

```bash
set +e
terraform plan -refresh-only -detailed-exitcode -input=false -lock=false -no-color
status=$?
set -e
case "$status" in
  0) echo "NO_DRIFT account=123456789012 workspace=production key=orders/production.tfstate" ;;
  2) echo "DRIFT_DETECTED owner=platform-oncall"; exit 2 ;;
  *) echo "DRIFT_CHECK_ERROR" >&2; exit 1 ;;
esac
```

Exit `0` is no drift, `2` is a real diff, and `1` is an error. The comparison is the named backend/workspace state versus the live provider API under the asserted read role. Platform on-call triages drift; any reconciliation is a separate reviewed plan/apply, never an automatic response to an unexplained diff.

> **Production:** do not automatically overwrite unexplained production drift. It may indicate an incident, an emergency repair, or a compromised identity — an automated "fix" at that point could erase the evidence.

Restrict direct console changes. When emergency manual changes occur, create the corresponding code change promptly so the declared configuration and the live state converge again.

---

## 6. Run Database Migrations Once

When every application replica runs its own migration at startup, a rolling deploy can start several replicas within the same few seconds. Each independently checks whether it's on the latest schema and, if not, tries to apply it — so two replicas can race to run the same `ALTER TABLE`, one blocking or erroring on the other's lock, while a third can pass its readiness check while connected against a schema that's only half-migrated. The failure isn't just "who runs it first" — it's that no single replica can prove the schema was migrated exactly once before any of them started serving traffic.

❌ Every application replica runs migrations at startup.

✅ A dedicated, serialized migration job runs with explicit identity and records the schema version, strictly before the application rollout that depends on the new schema begins.

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

[Alembic](https://alembic.sqlalchemy.org/) is the schema-migration runner invoked above; `upgrade "$TARGET_REVISION"` replays migration scripts up to that named revision, and `alembic current` reports the revision ID the database now believes it holds — a specific, comparable value, not just a process exit code.

> **Production:** a passing job is a match between three things, not one exit code. `alembic current` must print exactly `2026_07_29_01 (head)`; a domain check must confirm the new shape is actually queryable, for example `SELECT count(*) FROM orders WHERE given_name IS NOT NULL` returning a non-zero count once the expand step from section 7 has run; and the upgrade step must have produced no error. If `alembic current` reports an older revision, or omits `(head)`, the upgrade stopped partway through — treat that as a hard failure that blocks the application rollout, because a replica starting against a half-migrated schema is exactly the race this dedicated job exists to prevent.

Prefer short-lived database identity or a secret broker when the platform supports it. The migration role should have schema permissions that the application runtime role does not.

---

## 7. Use Expand and Contract

**Expand and contract** is a sequencing pattern that changes a schema across several small releases instead of one: it *expands* by adding the new shape while the old shape still works, migrates data and reads over subsequent releases, and only *contracts* by removing the old shape once nothing running still depends on it. Every release in between stays readable by whichever application version happens to be running.

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

Every intermediate state must support the versions that can coexist during rolling deployment and rollback — that's the same compatibility requirement [Deployment Strategies and Progressive Delivery](02_deployment_strategies.md) covers for application code, applied to the schema itself.

---

## 8. Design Backfills as Production Workloads

A **backfill** is the job that populates the newly added shape for rows that already existed before the expand release shipped — new writes get both the old and new fields going forward, but rows written earlier need a bulk job to catch up. That job runs against live production tables, competing with real traffic for locks, I/O, and connections, so it needs the same operational controls as any other production workload:

- **★ idempotent and resumable;**
- **★ partitioned into bounded batches;**
- **★ rate-limited;**
- **★ observable;**
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

The launch-blocking subset is those four starred controls plus single-run protection. A restart from checkpoint `customer_id=42000` reports `processed=500 next_checkpoint=42500 rate=198/s`; rerunning with the same checkpoint changes zero already-completed rows and advances deterministically.

> **Production:** do not hide a multi-hour backfill inside a deployment job timeout. Treat it as an operation with its own lifecycle and its own release dependency — the release that switches reads to the new field waits on the backfill's completion signal, not on a deploy step finishing.

---

## 9. Backups Are Not Rollbacks

> **Edge case:** read this before a high-risk migration, not during one — restore decisions made under incident pressure without this groundwork tend to be worse than the problem they're solving.

Before a high-risk migration:

- **★ verify backup recency and scope;**
- **★ test restore procedures;**
- **★ measure restore time against recovery objectives;**
- **★ know which later writes would be lost;**
- capture schema and migration version;
- **★ define who authorizes restore.**

Before launch, record `backup_id=orders-20260908T0200Z`, `scope=orders-prod`, `authorized_by=incident-commander`, and a tested restore result with elapsed time. A backup for another database, an expired recovery point, or missing authorization blocks restore before any live target is overwritten.

Restoring a database can be much more disruptive than rolling forward with a corrective migration, because a restore discards every write made since the backup, not just the migration's effects. Prefer compatibility and reversible steps over relying on restore.

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

A compromised application deployment can modify networking, data services, and **IAM** — Identity and Access Management, the cloud platform's system for what an identity is allowed to do — letting one compromised credential touch every layer at once. Split roles by operation.

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
