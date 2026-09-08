# Production CI/CD Notes

> Practical guidance for designing, securing, and operating production delivery pipelines with GitHub Actions.

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-production-2088FF.svg?logo=githubactions&logoColor=white)](https://docs.github.com/actions)
[![Docker](https://img.shields.io/badge/Docker-immutable_builds-2496ED.svg?logo=docker&logoColor=white)](https://docs.docker.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-delivery_patterns-326CE5.svg?logo=kubernetes&logoColor=white)](https://kubernetes.io/docs/)
[![AWS](https://img.shields.io/badge/AWS-OIDC_%26_ECS-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/Terraform-IaC-844FBA.svg?logo=terraform&logoColor=white)](https://developer.hashicorp.com/terraform)

---

## Structure

```text
cicd-notes/
│
│ ── FOUNDATIONS ───────────────────────────────────────────────
├── 01_fundamentals/
│   ├── 01_production_delivery_flow.md      End-to-end delivery model
│   ├── 02_branching_and_change_control.md  Trunk, rulesets, reviews, queues
│   └── 03_testing_and_quality_gates.md      Test layers and merge gates
│
│ ── GITHUB ACTIONS ────────────────────────────────────────────
├── 02_github_actions/
│   ├── 01_workflow_building_blocks.md      Events, jobs, data, and contexts
│   ├── 02_pull_request_ci.md                Secure, deterministic PR validation
│   ├── 03_build_publish_and_promote.md      Build once and promote by digest
│   ├── 04_reusable_workflows_and_actions.md Central automation contracts
│   ├── 05_cross_repository_orchestration.md Dispatch, GitOps, and sync patterns
│   ├── 06_performance_and_reliability.md    Matrix, cache, concurrency, cost
│   ├── 07_external_repository_integrations.md External App/service bindings
│   └── 08_cross_repository_contract_lifecycle.md Versioned contract lifecycle
│
│ ── SECURITY AND SUPPLY CHAIN ─────────────────────────────────
├── 03_security_and_supply_chain/
│   ├── 01_permissions_secrets_and_oidc.md  Short-lived, least-privilege access
│   ├── 02_workflow_and_runner_hardening.md Untrusted input and runner isolation
│   ├── 03_sbom_provenance_and_attestations.md Verifiable build provenance
│   ├── 04_dependency_update_governance.md Pin update governance
│   └── 05_github_app_provisioning.md App permissions, installation, and keys
│
│ ── DELIVERY OPERATIONS ───────────────────────────────────────
├── 04_delivery_operations/
│   ├── 01_environments_and_promotions.md    Deployment gates and promotion
│   ├── 02_deployment_strategies.md          Rolling, blue-green, canary, flags
│   ├── 03_infrastructure_and_database_changes.md IaC and safe migrations
│   └── 04_verification_observability_and_rollback.md Safe completion criteria
│
│ ── PRODUCTION REFERENCE ──────────────────────────────────────
└── 05_reference/
    ├── 01_provisioning_the_reference_platform.md Terraform for ECR, ECS, and IAM roles
    ├── 02_end_to_end_production_example.md  Docker service deployed to AWS ECS
    └── 03_production_readiness_checklist.md Review and adoption checklist
```

---

## Contents

### Foundations — [full index](01_fundamentals/README.md)

| Guide | Description |
|-------|-------------|
| [Production delivery flow](01_fundamentals/01_production_delivery_flow.md) | CI, continuous delivery, immutable artifacts, promotion, and feedback |
| [Branching and change control](01_fundamentals/02_branching_and_change_control.md) | Trunk-based work, rulesets, `CODEOWNERS`, merge queues, and releases |
| [Testing and quality gates](01_fundamentals/03_testing_and_quality_gates.md) | A risk-based test portfolio and required-check design |

### GitHub Actions — [full index](02_github_actions/README.md)

| Guide | Description |
|-------|-------------|
| [Workflow building blocks](02_github_actions/01_workflow_building_blocks.md) | Events, filters, jobs, expressions, outputs, artifacts, and environments |
| [Pull-request CI](02_github_actions/02_pull_request_ci.md) | A complete PR workflow with secure event handling |
| [Build, publish, and promote](02_github_actions/03_build_publish_and_promote.md) | Immutable images, digests, attestations, and promotion |
| [Reusable workflows and actions](02_github_actions/04_reusable_workflows_and_actions.md) | Choosing and governing reusable workflows, composite actions, and templates |
| [Cross-repository orchestration](02_github_actions/05_cross_repository_orchestration.md) | Dispatch APIs, shared workflows, GitOps, dependency updates, and synchronization |
| [Performance and reliability](02_github_actions/06_performance_and_reliability.md) | Concurrency, matrices, caches, timeouts, reruns, and cost |
| [External repository integrations](02_github_actions/07_external_repository_integrations.md) | Establish ownership, external bindings, explicit unknowns, and runtime evidence |
| [Cross-repository contract lifecycle](02_github_actions/08_cross_repository_contract_lifecycle.md) | Publish, consume, test, update, and deprecate shared libraries and schemas |

### Security and Supply Chain — [full index](03_security_and_supply_chain/README.md)

| Guide | Description |
|-------|-------------|
| [Permissions, secrets, and OIDC](03_security_and_supply_chain/01_permissions_secrets_and_oidc.md) | Token boundaries, GitHub Apps, cloud federation, and trust policies |
| [Workflow and runner hardening](03_security_and_supply_chain/02_workflow_and_runner_hardening.md) | SHA pinning, injection prevention, fork safety, and ephemeral runners |
| [SBOM, provenance, and attestations](03_security_and_supply_chain/03_sbom_provenance_and_attestations.md) | Producing and verifying supply-chain evidence |
| [Dependency update governance](03_security_and_supply_chain/04_dependency_update_governance.md) | Review tiering, required CI, CVE escalation, and staleness tracking for pin updates |
| [GitHub App provisioning](03_security_and_supply_chain/05_github_app_provisioning.md) | Registration permissions, installation scope, key rotation, audit identity, and revocation |

### Delivery Operations — [full index](04_delivery_operations/README.md)

| Guide | Description |
|-------|-------------|
| [Environments and promotions](04_delivery_operations/01_environments_and_promotions.md) | Staging and production controls without rebuilding |
| [Deployment strategies](04_delivery_operations/02_deployment_strategies.md) | Selecting rolling, blue-green, canary, and feature-flag techniques |
| [Infrastructure and database changes](04_delivery_operations/03_infrastructure_and_database_changes.md) | Terraform controls and expand-contract migrations |
| [Verification, observability, and rollback](04_delivery_operations/04_verification_observability_and_rollback.md) | Health gates, deployment telemetry, rollback, and DORA signals |
| [Configuration versioning and recovery](04_delivery_operations/05_configuration_versioning_and_recovery.md) | Versioning, validating, and restoring config and feature-flag state alongside a digest |

### Production Reference — [full index](05_reference/README.md)

| Guide | Description |
|-------|-------------|
| [Provisioning the reference platform](05_reference/01_provisioning_the_reference_platform.md) | Terraform for the ECR repository, ECS clusters/services, and IAM deployment roles the example below assumes |
| [End-to-end production example](05_reference/02_end_to_end_production_example.md) | A cohesive Python, Docker, ECR, and ECS delivery design |
| [Production-readiness checklist](05_reference/03_production_readiness_checklist.md) | A concise review checklist and maturity ladder |

---

## Reading Order

> [!TIP]
> Pick a path that matches the decision you need to make; the guides are also linked in a complete sequential path.

### New to CI/CD

1. [Production delivery flow](01_fundamentals/01_production_delivery_flow.md) — establish the delivery mental model
2. [Workflow building blocks](02_github_actions/01_workflow_building_blocks.md) — learn how GitHub Actions expresses that model
3. [Pull-request CI](02_github_actions/02_pull_request_ci.md) — implement the first practical gate

   **Stop here** if a merge gate is all you need right now. Continue only once you need to publish a deployable artifact and move it between environments.
4. [Build, publish, and promote](02_github_actions/03_build_publish_and_promote.md) — turn a passing PR into one immutable, digest-addressed release candidate
5. [Environments and promotions](04_delivery_operations/01_environments_and_promotions.md) — promote that digest through staging and production without rebuilding it

   Required production-identity continuation: read [Permissions, secrets, and OIDC](03_security_and_supply_chain/01_permissions_secrets_and_oidc.md) before this promotion touches real production credentials — the deployment role and trust policy it describes are inputs this step assumes.

### Build a Production Pipeline

1. [Branching and change control](01_fundamentals/02_branching_and_change_control.md) — establish ruleset and ownership evidence
2. [Testing and quality gates](01_fundamentals/03_testing_and_quality_gates.md) — define merge evidence before writing YAML
3. [Build, publish, and promote](02_github_actions/03_build_publish_and_promote.md) — create one immutable release candidate
4. [Permissions, secrets, and OIDC](03_security_and_supply_chain/01_permissions_secrets_and_oidc.md) — secure deployment identity

   **Milestone:** stop here if you need a registry-published, attested candidate with short-lived identity. Continue for the AWS-specific network/bootstrap prerequisites and ECS assembly.
5. [Provisioning the reference platform](05_reference/01_provisioning_the_reference_platform.md) — verify external owners, then create ECR, ECS, and IAM resources
6. [End-to-end production example](05_reference/02_end_to_end_production_example.md) — assemble the complete design

### Standardize Multiple Repositories

1. [Reusable workflows and actions](02_github_actions/04_reusable_workflows_and_actions.md) — centralize implementation safely
2. [Cross-repository orchestration](02_github_actions/05_cross_repository_orchestration.md) — select the correct coordination pattern
3. [External repository integrations](02_github_actions/07_external_repository_integrations.md) — verify App/service-side access and runtime consumption

   **Milestone:** stop here once the coordination contract and its external access/runtime evidence are verified. Continue for security hardening and adoption audit.
4. [Workflow and runner hardening](03_security_and_supply_chain/02_workflow_and_runner_hardening.md) — protect the larger trust boundary
5. [Production-readiness checklist](05_reference/03_production_readiness_checklist.md) — audit adoption consistently

### Improve an Existing Pipeline

1. [Performance and reliability](02_github_actions/06_performance_and_reliability.md) — reduce latency, waste, and race conditions

   **Milestone:** stop here when the measured queue/critical-path bottleneck is fixed. Continue when rollout risk, artifact trust, or recovery is the next constraint.
2. [Deployment strategies](04_delivery_operations/02_deployment_strategies.md) — reduce release blast radius
3. [SBOM, provenance, and attestations](03_security_and_supply_chain/03_sbom_provenance_and_attestations.md) — make released artifacts verifiable
4. [Verification, observability, and rollback](04_delivery_operations/04_verification_observability_and_rollback.md) — close the operational feedback loop with signer-aware recovery
