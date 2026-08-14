# Production Reference

> An architecture walkthrough of the repository's production techniques assembled into one design, and a checklist for reviewing real delivery systems.

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-reference-2088FF.svg?logo=githubactions&logoColor=white)](https://docs.github.com/actions)
[![Docker](https://img.shields.io/badge/Docker-image-2496ED.svg?logo=docker&logoColor=white)](https://docs.docker.com/)
[![AWS](https://img.shields.io/badge/AWS-ECS-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_provisioning_the_reference_platform.md](01_provisioning_the_reference_platform.md) | Platform bootstrap | Terraform for the ECR repository, ECS clusters/services, IAM deployment roles, and first task-definition revision the example below assumes |
| [02_end_to_end_production_example.md](02_end_to_end_production_example.md) | Reference pipeline | A Python and Docker service built once and promoted to AWS ECS |
| [03_production_readiness_checklist.md](03_production_readiness_checklist.md) | Review checklist | Controls, questions, and a practical maturity model |

---

## Reading Order

1. **Provisioning the reference platform** — create the ECR repository, ECS clusters and services, IAM deployment roles, and first task-definition revision the example below assumes already exist
2. **End-to-end production example** — walk the techniques assembled into one architecture, traced end to end by digest and task-definition revision rather than run as a standalone repository
3. **Production-readiness checklist** — review a pipeline or plan incremental adoption

---

## Prerequisites

- [GitHub Actions](../02_github_actions/README.md)
- [Delivery Operations](../04_delivery_operations/README.md)
