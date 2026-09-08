# Production CI/CD as a Delivery System

> **Who this is for**: Engineers designing or reviewing a delivery pipeline. Basic Git and automated-testing knowledge is assumed.

## The short version

A green pipeline only proves its commands exited successfully — not that the release is safe to run in production. **CI/CD** — continuous integration and continuous delivery/deployment, the automated path from a source change to a running production system — closes that gap by producing evidence: one immutable build, identified by a content **digest** (a hash of the artifact's exact bytes, so two builds count as the same release only when their digest matches), that staging verifies and production runs unchanged. Because staging and production run the identical digest, a passing staging check is evidence about the bytes production will actually execute, not about a separate rebuild that merely resembles it.

**What you need (4 things):**

1. A source commit to build from.
2. A build step that produces one immutable artifact and records its content digest.
3. Two or more environments that deploy that same digest — never a fresh rebuild.
4. A verification step in each environment that gates promotion to the next one.

**Worked example:**

```text
commit 8f31c2a7d9 merges to main
        │
        ▼
build job runs once, producing:
   tag:    ghcr.io/acme/orders:git-8f31c2a   (convenience label — can move)
   digest: sha256:4ae0...9c1d                (content hash — the real identity)
        │
        ▼
staging deploys digest sha256:4ae0...9c1d
        │  smoke tests pass
        ▼
production deploys digest sha256:4ae0...9c1d
        (same bytes staging just tested — not a fresh rebuild)
```

**Success signal:** the staging and production deployment records cite the identical digest (`sha256:4ae0...9c1d`). If they differ, production is running bytes staging never tested.

**Not handled yet:** [scanning, SBOM/provenance, and rollout strategy](#4-hardening-the-baseline-what-production-adds) that harden this baseline, [why schema and config rollback depend on binary compatibility](#5-rollback-reality-differs-by-change-type), [which pipeline principles matter from day one](#6-a-few-principles-do-most-of-the-work), and [common failure modes this baseline doesn't prevent by itself](#7-what-breaks-in-practice-and-the-tell-youll-see).

---

## 1. CI/CD's Job Is to Produce Evidence, Not a Green Checkmark

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

The evidence does not all live in Git. Keep an evidence map so a checked-in declaration is not mistaken for an enforced control:

| Claim | Owner and authoritative inspection surface | Repository proves | Unknown without control-plane access | Proof it acted |
|---|---|---|---|---|
| `required-ci` validates the candidate | Repository; `.github/workflows/ci.yml` | Trigger, job graph, and declared check name | Which active GitHub ruleset targets `main`, its source restriction, and bypass actors | A deliberately failing PR produces a red `required-ci` from the expected GitHub App and GitHub blocks merge |
| Production requires approval | GitHub environment administrator; environment settings/API | The deploy job names `environment: production` | Reviewer list, allowed refs, custom gates, self-review, and admin bypass | An unapproved run remains waiting and receives no production credential |

Record inaccessible fields as `unknown`; repository silence is not evidence that an external control is absent or correctly configured.

*Synthetic* signals here mean scripted requests that continuously exercise key user journeys — placing an order, logging in — independent of real traffic, so a broken flow is caught even during a quiet period.

> **Key insight**: A green workflow only proves that its declared commands exited successfully. The pipeline is trustworthy only when those commands produce the evidence the business risk requires.

---

## 2. Which Boundary You Automate Depends on Risk, Not Maturity

These terms describe different automation boundaries:

| Practice | Automated boundary | Human decision |
|----------|--------------------|----------------|
| **Continuous integration** | Change validation and integration into the main line | Whether the change is approved and merged |
| **Continuous delivery** | A release candidate is always deployable; staging is commonly automatic | Whether or when to promote to production |
| **Continuous deployment** | Every qualifying change is promoted automatically | Policy defines exceptions rather than approving every release |

A **release candidate** is a specific built artifact considered feature-complete and eligible to ship, pending verification — the same one every environment deploys, never a fresh build per environment.

The same repository can use different models by risk:

```text
documentation     → continuous deployment
internal service  → automated production after health gates
payment service   → continuous delivery with production approval
database rewrite  → scheduled change with an explicit runbook
```

> **Edge case:** some changes don't fit any automated boundary at all — a cross-shard data migration or an irreversible schema rewrite gets a scheduled change with a runbook instead of a pipeline stage, because the risk lives in the change itself, not in how it's validated.

Manual approval is not inherently safer. An approval that merely repeats "CI is green" adds latency without new evidence. Useful approvals resolve information automation does not have: change timing, business coordination, incident state, or exceptional risk.

---

## 3. Build the Artifact Once; Promote the Same Digest Everywhere

> **Core:** this is the mechanism every other section in this file either hardens or assumes. Once you can restate why staging and production must deploy identical bytes, you own the baseline.

The build output is the release candidate from [section 2](#2-which-boundary-you-automate-depends-on-risk-not-maturity) — the same digest staging just verified, deployed to production without a rebuild:

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

This is the **Immutability** principle from [section 6](#6-a-few-principles-do-most-of-the-work) already in practice: the digest never changes between environments, only the traffic pointed at it does.

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

This record is what **Auditability** ([section 6](#6-a-few-principles-do-most-of-the-work)) looks like in practice: source commit, artifact, digest, run, and target are all linked in one place, so "what's running in production and why" is a lookup, not an investigation.

---

## 4. Hardening the Baseline: What Production Adds

> **Production:** everything below wraps the build-once/promote-many mechanism from [section 3](#3-build-the-artifact-once-promote-the-same-digest-everywhere) with checks and rollout controls. None of it changes that mechanism; skip this section until you're taking real production traffic.

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

Each addition earns its place by closing one gap the baseline leaves open:

- **IaC** — infrastructure as code: infrastructure changes expressed as versioned, reviewable configuration (Terraform, CloudFormation, Pulumi) instead of manual console clicks. Validating it in the pull request catches a bad infrastructure change in review instead of mid-deploy.
- **SBOM** — software bill of materials: a generated inventory of every package and version inside the artifact. When a dependency is disclosed as vulnerable, you grep the SBOM for it instead of re-scanning every running system from scratch.
- **Provenance** — a signed record of which source commit, workflow run, and builder produced the artifact, so a consumer can verify the digest actually came from your pipeline and wasn't substituted afterward.
- **Observability gate** — an automated check that blocks promotion unless post-deploy metrics (error rate, latency, saturation) stay inside an agreed band for a bake period, catching regressions that pass tests but misbehave only under real traffic.
- **Blue-green rollout** — running the new version fully alongside the old one and switching traffic in one cutover; rollback means pointing traffic back, not redeploying.
- **Canary rollout** — sending a small slice of real traffic to the new version before the rest, so a bad release is caught while it's only hurting that slice.
- **Merge queue** — a serialization mechanism that tests and merges candidate changes one at a time against a moving main, so two individually passing pull requests can't combine into a broken main; see [Branching and Change Control](02_branching_and_change_control.md) for the full mechanism.

This is a baseline hardening set, not a mandate. A static website may need fewer stages; a regulated or stateful system may need more.

---

## 5. Rollback Reality Differs by Change Type

A release drops a column its new code no longer needs and deploys the new binary. Autoscaling hasn't cycled every pod yet, though — some old-binary instances are still serving requests, still reading that column on every one of them. They now get `null` where they expect structured data and start erroring on a path nobody staged for. The runbook says "redeploy the previous image" to roll back, but the column is already gone: the old binary now fails the same way the new one did, and the actual fix is restoring the table from a backup, not reverting a deploy.

That's why application, infrastructure, and data changes should coordinate but are never one reversible operation — each has a different rollback reality, and a plan that only reverts code doesn't survive a change that also reverts data:

| Change | Typical tool | Rollback reality |
|--------|--------------|------------------|
| Stateless application | Container or package deployment | Often redeploy the previous digest |
| Infrastructure | Terraform, CloudFormation, Pulumi | Reverse plans can be destructive or incomplete |
| Database schema | Alembic, Flyway, Liquibase | Destructive migrations may be irreversible |
| Feature exposure | Feature-flag platform | Often the fastest operational kill switch |
| Configuration | GitOps or parameter store | Roll back only if old and new binaries both tolerate it |

**GitOps** — keeping a system's desired state as declarative configuration committed to Git, with a controller continuously reconciling the live system to match it — makes reverting a *config* change mechanically trivial: revert the commit, and the controller re-applies the old state on its own. But that revert only helps if the binary currently running can still parse the old shape. If the new binary dropped a field the old one required, or now requires a field the old configuration never set, the reconciled "old" state is one the running process still can't use — the revert succeeds and the outage continues.

The sequence below is how the column-drop failure above gets avoided: it never lets the schema and the code that depends on it change in the same step.

```text
expand schema
    → deploy code compatible with old and new schema
    → backfill (copy existing rows into the new representation in bounded batches)
    → switch reads
    → observe
    → contract schema in a later release (remove the old field only after no old binary needs it)
```

**Success signal:** at every phase, the previous release's code keeps running correctly against the current schema — you could pause between phases indefinitely without an outage. **Silent-failure tell:** a migration that looks clean in the deploy log but starts erroring only after autoscaling replaces the last old pod — evidence the expand phase was never actually backward-compatible, just untested against overlap.

See [Infrastructure and Database Changes](../04_delivery_operations/03_infrastructure_and_database_changes.md) for the full pattern.

---

## 6. A Few Principles Do Most of the Work

Ten principles show up across mature pipelines, but you don't need all ten on day one. In practice you'll lean on **Immutability** and **Auditability** (★) from the very first pipeline you build — both are already at work in the [build-once/promote-many trace](#3-build-the-artifact-once-promote-the-same-digest-everywhere) above: the digest never changes between environments (Immutability), and the promotion record links source, digest, and target (Auditability). The rest earn their keep as you add stages.

| Principle | Practical consequence |
|-----------|-----------------------|
| Fast feedback first | Cheap lint and unit checks fail before expensive builds |
| Determinism | Lock dependencies and declare tool versions |
| Least privilege | Read-only is the workflow default; write access is job-specific |
| **★ Immutability** | Promote identifiers, never rebuild a tested release |
| Idempotency | Retrying a deploy converges instead of duplicating resources |
| Bounded execution | Every external call and job has a timeout |
| Serialized state change | One production deploy or migration per target |
| Observable outcomes | Deployment metadata is joined to runtime signals |
| Recoverability | Previous versions and recovery commands are retained and tested |
| **★ Auditability** | Source, approvals, identity, artifact, target, and result are linked |

A team notices their pipeline takes 40 minutes and cuts it to 12 by dropping the integration suite and shortening the observability gate's bake time from 30 minutes to 2. Workflow duration improves — and so does the odds that a regression only visible under real traffic reaches production before anyone is watching for it. The 2-minute bake window doesn't prove the release is healthy; it proves the metrics hadn't diverged yet.

> **Principle**: Optimize the path to trustworthy feedback, not merely workflow duration.

---

## 7. What Breaks in Practice, and the Tell You'll See

**Workflow success is mistaken for release success**

The deployment command returns before the platform reaches steady state. Wait for platform convergence, then run health and synthetic checks.

**Staging is not representative**

Different identity policies, networking, data scale, or feature flags make the staging result weak evidence. Document intentional differences and test production-specific controls without exposing users.

**Every pipeline becomes unique**

Copied workflows drift. Centralize stable contracts with reusable workflows while keeping service-specific commands in the service repository.

**The pipeline owns too much**

A single workflow applies infrastructure, migrates data, deploys code, changes traffic, and deletes old resources. Split operations by permission and recovery boundary, then coordinate them explicitly.

**Rollback exists only as a command**

The previous binary cannot run against the new schema, or the old configuration has been deleted. Recovery must be compatibility-tested, not merely documented — see [section 5](#5-rollback-reality-differs-by-change-type) for what that testing looks like.

---

## 8. References

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts)
- [Deploying with GitHub Actions](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments)

---

**Next**: [Branching and Change Control](02_branching_and_change_control.md)
