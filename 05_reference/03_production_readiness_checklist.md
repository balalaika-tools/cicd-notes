# CI/CD Production-Readiness Checklist

> **Who this is for**: Engineers reviewing an existing pipeline or planning incremental production hardening. Read the [End-to-End Production Example](02_end_to_end_production_example.md) for context.

---

## 1. Source and Change Control

- [ ] The default branch is protected by a ruleset or branch protection.
- [ ] Direct pushes, force pushes, and deletion are blocked.
- [ ] Pull requests require independent review.
- [ ] Sensitive paths have current `CODEOWNERS`.
- [ ] Workflow, infrastructure, migration, and security changes require appropriate owners.
- [ ] Approvals become stale after material changes.
- [ ] Required-check names are stable and documented.
- [ ] A merge queue is enabled where integration races are common.
- [ ] Required workflows listen for `merge_group` when a merge queue is used.
- [ ] Bypass access is narrow, audited, and supported by an emergency procedure.
- [ ] Changes are small enough to review and release independently.
- [ ] Incomplete behavior is isolated behind a controlled feature flag.

---

## 2. Pull-Request CI

- [ ] The workflow uses `pull_request`, not privileged `pull_request_target`, for code validation.
- [ ] The default `GITHUB_TOKEN` permission is `contents: read` or narrower.
- [ ] Forked PRs do not receive secrets or privileged self-hosted runners.
- [ ] Formatting, linting, compilation/types, and unit tests run.
- [ ] Relevant component, contract, and migration tests run.
- [ ] Dependency, secret, static-code, IaC, and container policy checks are defined.
- [ ] Security findings have severity, ownership, exception, and expiry policy.
- [ ] Dependencies and toolchains are locked.
- [ ] The build works without a cache.
- [ ] A stable aggregator job reports the final required result.
- [ ] Path filtering includes changed dependents and cannot omit the required check.
- [ ] A scheduled full build validates change-detection logic.
- [ ] Flaky tests are measured, owned, and time-bounded.
- [ ] Test reports and failure diagnostics have deliberate retention.

---

## 3. Workflow Security

- [ ] External actions and reusable workflows are pinned to reviewed full commit SHAs.
- [ ] Update automation proposes pinned dependency revisions.
- [ ] Organization/repository policy restricts unapproved actions.
- [ ] Untrusted context values are passed as data, not interpolated into shell code.
- [ ] Branch names, dispatch inputs, titles, bodies, and artifact content are treated as untrusted.
- [ ] Checkout does not persist credentials when Git writes are unnecessary.
- [ ] Privileged jobs do not execute pull-request code.
- [ ] Low-trust artifacts are validated before privileged consumption.
- [ ] Workflow changes are protected by code owners.
- [ ] No workflow uses `permissions: write-all`.
- [ ] Logs, caches, artifacts, and debug bundles are reviewed for secret exposure.
- [ ] A workflow-compromise response procedure exists.

---

## 4. Identity and Secrets

- [ ] Each job declares only the GitHub token permissions it needs.
- [ ] Cross-repository GitHub access uses a narrowly installed GitHub App.
- [ ] Fine-grained PATs have owners, expiry, repository scope, and rotation if still required.
- [ ] Cloud access uses OIDC and short-lived credentials.
- [ ] Cloud trust restricts audience and subject to the intended repository and environment/ref.
- [ ] OIDC trust matches the repository's actual name-based or immutable-ID subject format.
- [ ] Environment branch/tag rules compensate when the OIDC subject uses an environment.
- [ ] Cloud permission policies are resource- and action-scoped.
- [ ] Build, staging, production, infrastructure, and migration roles are distinct.
- [ ] Environment secrets are released only after protection rules.
- [ ] Static secrets are stored at the narrowest level and rotated.
- [ ] No secret is passed in a job output, cache, artifact, or command line without necessity.
- [ ] GitHub App private keys are not available to every repository by default.

---

## 5. Build and Supply Chain

- [ ] Accepted source is rebuilt in a trusted main-line workflow.
- [ ] The artifact is built once per release candidate.
- [ ] Dependencies, base images, tools, actions, and build configuration are constrained.
- [ ] Container images run as a non-root user where practical.
- [ ] A registry immutability or equivalent retention policy exists.
- [ ] The artifact digest is recorded and used as deployment identity.
- [ ] Staging and production use the same digest.
- [ ] Release metadata links source SHA, workflow run, artifact, and digest.
- [ ] An SBOM describes the final artifact, not only the source tree.
- [ ] The final artifact is scanned.
- [ ] Provenance is generated for released artifacts.
- [ ] Consumers verify provenance against repository/workflow policy.
- [ ] Registry rescanning finds vulnerabilities discovered after build.
- [ ] Deployed inventory can answer which digests contain an affected package.
- [ ] Multi-artifact releases use a versioned manifest.

---

## 6. Reuse and Cross-Repository Design

- [ ] Shared implementation uses reusable workflows or versioned tools rather than copied YAML.
- [ ] Reusable-workflow inputs form a small, typed, validated contract.
- [ ] Callers pin central workflows and receive update PRs.
- [ ] Nested workflow permissions cannot and do not need to elevate.
- [ ] Named secrets are passed instead of broad inheritance.
- [ ] Central workflow adoption and failures are observable.
- [ ] Cross-repository automation uses the correct pattern: reuse, dispatch, GitOps, dependency, sync, or polling.
- [ ] Dispatch payloads carry immutable artifact identity and correlation metadata.
- [ ] The receiver validates payload schema, source allowlist, and artifact provenance.
- [ ] Dispatch acceptance is not mistaken for downstream success.
- [ ] Cross-repository operations are idempotent.
- [ ] Event loops are prevented.
- [ ] Shared libraries and schemas are versioned and updated through compatibility-tested PRs.
- [ ] File synchronization opens reviewable PRs rather than pushing destination `main`.

---

## 7. Environments and Promotion

- [ ] GitHub environments represent real deployment policy boundaries.
- [ ] Staging deploys automatically from a trusted release source where appropriate.
- [ ] Production permits only intended branches, tags, and workflows.
- [ ] Manual approval adds information not already provided by CI.
- [ ] Self-approval is prevented for independent production gates.
- [ ] Administrator bypass is disabled or strictly audited.
- [ ] Plan limitations do not leave private repositories without the expected protection.
- [ ] Promotion revalidates artifact identity and evidence.
- [ ] Long-lived release candidates have durable records independent of one workflow run.
- [ ] Runtime configuration is environment-specific; build output is not.
- [ ] Environment differences and their compensating tests are documented.
- [ ] Preview environments are isolated, labeled, cost-bounded, and automatically expired.

---

## 8. Deployment Safety

- [ ] Production deployments are serialized per target.
- [ ] The team understands that GitHub concurrency is not a durable FIFO queue.
- [ ] The rollout strategy matches state, capacity, routing, and observability constraints.
- [ ] Old and new versions are API, configuration, and data compatible during overlap.
- [ ] Readiness, liveness, and startup checks have distinct semantics.
- [ ] Connections drain before old instances terminate.
- [ ] Deployment operations have timeouts.
- [ ] Progressive rollout gates use version-segmented metrics and minimum samples.
- [ ] Missing telemetry stops risky promotion.
- [ ] Feature flags have owners, defaults, audit, and removal dates.
- [ ] Shadow traffic cannot produce unsafe side effects.
- [ ] Platform circuit breakers and application-level alarms are configured.

---

## 9. Infrastructure and Database

- [ ] Infrastructure plans run with controlled credentials.
- [ ] Pull-request plans are treated as speculative unless the exact plan is preserved and approved.
- [ ] The final applied Terraform plan is explicitly approved or policy-authorized.
- [ ] Saved plans and working directories are protected as sensitive artifacts.
- [ ] Remote state and state locking are enabled.
- [ ] CI concurrency aligns with each state/workspace.
- [ ] Drift detection runs and unexpected drift is investigated.
- [ ] Infrastructure, application, and migration permissions are separate.
- [ ] Migrations run once as a dedicated serialized operation.
- [ ] Schema changes use expand-contract.
- [ ] Backfills are bounded, resumable, rate-limited, and observable.
- [ ] Lock and statement timeouts protect the database.
- [ ] Restore has been tested against recovery objectives.
- [ ] Old binaries remain compatible until the rollback window closes.
- [ ] Destructive cleanup occurs in a later release.

---

## 10. Verification, Observability, and Recovery

- [ ] Deployment waits for platform convergence.
- [ ] Smoke tests verify the expected release identity.
- [ ] Critical dependencies and business paths are exercised.
- [ ] Production synthetic tests use safe identities and idempotent data.
- [ ] Deployment markers link runtime telemetry to source and digest.
- [ ] Logs, traces, errors, and deployment records include bounded release identity.
- [ ] Error, latency, saturation, SLO, and business signals are evaluated.
- [ ] Automatic rollback signals are reliable and bounded.
- [ ] The previous digest, task definition, configuration, flags, and migration state are retained.
- [ ] Rollback deploys the previous immutable artifact rather than rebuilding source.
- [ ] Roll-forward is documented for irreversible state changes.
- [ ] Recovery workflows use the same identity and environment controls as releases.
- [ ] Rollback and restore are exercised in non-production or game days.
- [ ] Delivery metrics are used to improve the system rather than rank individuals.
- [ ] Deployment incidents feed back into tests, rollout, observability, or architecture.

---

## 11. Performance and Cost

- [ ] Queue time, setup time, command time, and critical path are measured separately.
- [ ] PR runs cancel when superseded.
- [ ] Matrices cover supported combinations without an unnecessary Cartesian product.
- [ ] `max-parallel` protects runners and rate-limited dependencies.
- [ ] Cache keys include lockfiles and relevant tool/runtime versions.
- [ ] Caches contain no secrets or release identities.
- [ ] External calls use bounded retries, backoff, and operation timeouts.
- [ ] Retry is limited to idempotent operations.
- [ ] Artifact retention matches investigation and audit needs.
- [ ] Release artifacts live in durable registries, not only workflow storage.
- [ ] Preview and temporary-resource cleanup has both event and scheduled reconciliation.
- [ ] Self-hosted runner queue time, patching, capacity, and external log forwarding are monitored.
- [ ] CI cost is attributable by repository, workflow, and major job.

---

## 12. Maturity Ladder

### Level 1 — Repeatable

```text
pull requests
required lint and tests
protected main
versioned workflow files
automated build
```

### Level 2 — Controlled

```text
immutable artifact digest
staging promotion
production environment gate
least-privilege permissions
serialized deployment
smoke verification
documented rollback
```

### Level 3 — Hardened

```text
OIDC cloud access
full-SHA action pinning
GitHub App cross-repository identity
ephemeral privileged runners
SBOM + provenance + verification
expand-contract migrations
tested recovery
```

### Level 4 — Progressive

```text
automated policy gates
canary or blue-green delivery
SLO-aware rollout
versioned cross-repository contracts
deployed-artifact inventory
continuous drift and vulnerability response
delivery metrics drive improvement
```

Do not jump levels by adding tools. Each level is achieved when the controls are routinely used, observable, and trusted during urgent work.

---

## 13. Recommended First Ten Improvements

For a typical repository starting with basic Actions:

1. Protect `main` with review and one stable required check.
2. Set workflow token permissions to `contents: read`.
3. Pin external actions and automate their updates.
4. Lock dependencies and make PR CI deterministic.
5. Build one artifact and record its digest.
6. Add staging deployment and bounded smoke verification.
7. Replace cloud access keys with OIDC and narrow roles.
8. Protect production with an environment and serialize deployment.
9. Retain and test a previous-digest recovery path.
10. Add SBOM, provenance, and verification at the promotion boundary.

---

**Next**: [Return to the Production CI/CD Notes index](../README.md)
