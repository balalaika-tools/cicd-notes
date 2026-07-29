# Production CI/CD as a Delivery System

> **Who this is for**: Engineers designing or reviewing a delivery pipeline. Basic Git and automated-testing knowledge is assumed.

---

## 1. The Delivery Contract

CI/CD is not a collection of workflow files. It is a controlled path that turns a source change into production evidence.

```text
Change proposed
      │
      ▼
┌───────────────┐    evidence     ┌────────────────┐
│ Pull-request  │───────────────> │ Merge decision │
│ validation    │                 └───────┬────────┘
└───────────────┘                         │
                                          ▼
                                ┌──────────────────┐
                                │ Immutable build  │
                                │ digest + metadata│
                                └────────┬─────────┘
                                         │ promote
                         ┌───────────────┴───────────────┐
                         ▼                               ▼
                  ┌─────────────┐                  ┌─────────────┐
                  │   Staging   │── verification ─>│ Production  │
                  └─────────────┘                  └──────┬──────┘
                                                          │
                                                          ▼
                                                  observe or recover
```

The delivery contract should answer:

| Question | Required evidence |
|----------|-------------------|
| Is the change reviewable? | Small diff, owner approval, linked intent |
| Is the source acceptable? | Lint, types, tests, security and policy checks |
| What exactly will run? | Immutable artifact digest tied to a commit |
| Who authorized production? | Ruleset, environment gate, or policy decision |
| Did the release work? | Health, smoke, synthetic, and business signals |
| How do we recover? | Previous digest, compatible data, and a tested procedure |

> **Key insight**: A green workflow only proves that its declared commands exited successfully. The pipeline is trustworthy only when those commands produce the evidence the business risk requires.

---

## 2. CI, Continuous Delivery, and Continuous Deployment

These terms describe different automation boundaries:

| Practice | Automated boundary | Human decision |
|----------|--------------------|----------------|
| **Continuous integration** | Change validation and integration into the main line | Whether the change is approved and merged |
| **Continuous delivery** | A release candidate is always deployable; staging is commonly automatic | Whether or when to promote to production |
| **Continuous deployment** | Every qualifying change is promoted automatically | Policy defines exceptions rather than approving every release |

The same repository can use different models by risk:

```text
documentation     → continuous deployment
internal service  → automated production after health gates
payment service   → continuous delivery with production approval
database rewrite  → scheduled change with an explicit runbook
```

Manual approval is not inherently safer. An approval that merely repeats “CI is green” adds latency without new evidence. Useful approvals resolve information automation does not have: change timing, business coordination, incident state, or exceptional risk.

---

## 3. A Mature Default Flow

```text
Feature branch
   ↓
Pull request
   ├── formatting, linting, and types
   ├── unit and contract tests
   ├── dependency, secret, and code scanning
   ├── container and IaC validation
   └── required review and ownership checks
   ↓
Merge queue or protected merge into main
   ↓
Build one immutable artifact
   ├── tag with commit SHA for discoverability
   ├── record the content digest as identity
   ├── generate SBOM and provenance
   └── publish to a trusted registry
   ↓
Deploy the digest to staging
   ├── migrations as a serialized operation
   ├── integration and smoke tests
   └── observability gate
   ↓
Promote the same digest
   ├── production environment policy
   ├── short-lived deployment identity
   └── rolling, blue-green, or canary rollout
   ↓
Verify service and business health
   ├── mark deployment successful
   └── halt, roll back, or roll forward
```

This is a baseline, not a mandate. A static website may need fewer stages; a regulated or stateful system may need more.

---

## 4. Build Once, Promote Many

The build output is a **release candidate**, not an environment-specific rebuild.

```text
source commit 8f31c2a
        │
        ▼
image tag:    ghcr.io/acme/orders:git-8f31c2a
image digest: sha256:4ae0...9c1d
        │
        ├── staging deploys sha256:4ae0...9c1d
        └── production deploys sha256:4ae0...9c1d
```

Tags are convenient references and can move. Digests identify content.

✅ Environment differences belong in runtime configuration, secret managers, feature flags, and infrastructure configuration.

❌ Rebuilding for production can introduce a different base image, dependency, compiler output, or timestamp from the version tested in staging.

A promotion record should carry:

```json
{
  "repository": "acme/orders",
  "source_sha": "8f31c2a7d9...",
  "artifact": "ghcr.io/acme/orders",
  "digest": "sha256:4ae0...9c1d",
  "build_run_id": "8912345678",
  "target_environment": "production"
}
```

---

## 5. Separate Concerns with Different Failure Properties

Application, infrastructure, and data changes should coordinate, but they should not be treated as one reversible operation.

| Change | Typical tool | Rollback reality |
|--------|--------------|------------------|
| Stateless application | Container or package deployment | Often redeploy the previous digest |
| Infrastructure | Terraform, CloudFormation, Pulumi | Reverse plans can be destructive or incomplete |
| Database schema | Alembic, Flyway, Liquibase | Destructive migrations may be irreversible |
| Feature exposure | Feature-flag platform | Often the fastest operational kill switch |
| Configuration | GitOps or parameter store | Roll back only if old and new binaries both tolerate it |

The deployment sequence must be compatibility-aware:

```text
expand schema
    → deploy code compatible with old and new schema
    → backfill
    → switch reads
    → observe
    → contract schema in a later release
```

See [Infrastructure and Database Changes](../04_delivery_operations/03_infrastructure_and_database_changes.md) for the full pattern.

---

## 6. Pipeline Design Principles

| Principle | Practical consequence |
|-----------|-----------------------|
| Fast feedback first | Cheap lint and unit checks fail before expensive builds |
| Determinism | Lock dependencies and declare tool versions |
| Least privilege | Read-only is the workflow default; write access is job-specific |
| Immutability | Promote identifiers, never rebuild a tested release |
| Idempotency | Retrying a deploy converges instead of duplicating resources |
| Bounded execution | Every external call and job has a timeout |
| Serialized state change | One production deploy or migration per target |
| Observable outcomes | Deployment metadata is joined to runtime signals |
| Recoverability | Previous versions and recovery commands are retained and tested |
| Auditability | Source, approvals, identity, artifact, target, and result are linked |

> **Principle**: Optimize the path to trustworthy feedback, not merely workflow duration.

---

## 7. Common Failure Modes

**Workflow success is mistaken for release success**

The deployment command returns before the platform reaches steady state. Wait for platform convergence, then run health and synthetic checks.

**Staging is not representative**

Different identity policies, networking, data scale, or feature flags make the staging result weak evidence. Document intentional differences and test production-specific controls without exposing users.

**Every pipeline becomes unique**

Copied workflows drift. Centralize stable contracts with reusable workflows while keeping service-specific commands in the service repository.

**The pipeline owns too much**

A single workflow applies infrastructure, migrates data, deploys code, changes traffic, and deletes old resources. Split operations by permission and recovery boundary, then coordinate them explicitly.

**Rollback exists only as a command**

The previous binary cannot run against the new schema, or the old configuration has been deleted. Recovery must be compatibility-tested, not merely documented.

---

## 8. References

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts)
- [Deploying with GitHub Actions](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments)

---

**Next**: [Branching and Change Control](02_branching_and_change_control.md)
