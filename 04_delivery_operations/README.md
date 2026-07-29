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

---

## Reading Order

1. **Environments and promotions** — define safe movement between targets
2. **Deployment strategies** — control production exposure
3. **Infrastructure and database changes** — handle changes with different rollback properties
4. **Verification, observability, and rollback** — prove and monitor the outcome

---

## Prerequisites

- [Build, publish, and promote](../02_github_actions/03_build_publish_and_promote.md)
- [Security and Supply Chain](../03_security_and_supply_chain/README.md)
