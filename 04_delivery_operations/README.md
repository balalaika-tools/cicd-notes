# Delivery Operations

> Techniques for promoting, releasing, observing, and recovering production changes.

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-deployments-2088FF.svg?logo=githubactions&logoColor=white)](https://docs.github.com/actions)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-rollouts-326CE5.svg?logo=kubernetes&logoColor=white)](https://kubernetes.io/docs/)
[![Terraform](https://img.shields.io/badge/Terraform-change_control-844FBA.svg?logo=terraform&logoColor=white)](https://developer.hashicorp.com/terraform)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_environments_and_promotions.md](01_environments_and_promotions.md) | Promotion | Environment protection, approvals, deployment identity, and artifact promotion |
| [02_deployment_strategies.md](02_deployment_strategies.md) | Rollout | Rolling, blue-green, canary, and feature-flag strategies |
| [03_infrastructure_and_database_changes.md](03_infrastructure_and_database_changes.md) | Stateful change | Terraform workflows and expand-contract migrations |
| [04_verification_observability_and_rollback.md](04_verification_observability_and_rollback.md) | Operations | Health gates, telemetry, rollback, and delivery metrics |
| [05_configuration_versioning_and_recovery.md](05_configuration_versioning_and_recovery.md) | Operations | Versioning, validating, and restoring config and feature-flag state alongside a digest |

---

## Reading Order

1. **Environments and promotions** — define safe movement between targets and produce one verified staged release
2. **Verification, observability, and rollback** — confirm that release is actually healthy before anything else builds on it

   **Milestone**: you can now promote a digest to production and prove it's serving correctly. **Stop here** if that closed loop is all you need. Continue when a change is too risky to expose to every user at once, touches infrastructure or a database, or when you need full incident recovery beyond a single rollback.
3. **Deployment strategies** — control production exposure for riskier changes
4. **Infrastructure and database changes** — handle changes with different rollback properties
5. **Configuration versioning and recovery** — make sure rollback restores the paired configuration and feature-flag state, not just the previous binary

---

## Prerequisites

- [Build, publish, and promote](../02_github_actions/03_build_publish_and_promote.md)
- [Reusable workflows and actions](../02_github_actions/04_reusable_workflows_and_actions.md) — the first promotion example calls a reusable deployment workflow; know its caller/callee contract first
- [Security and Supply Chain](../03_security_and_supply_chain/README.md)
