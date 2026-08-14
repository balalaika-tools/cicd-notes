# GitHub Actions

> Workflow mechanics and production patterns for CI, immutable builds, reuse, cross-repository coordination, and reliable execution.

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-workflows-2088FF.svg?logo=githubactions&logoColor=white)](https://docs.github.com/actions)
[![Docker](https://img.shields.io/badge/Docker-builds-2496ED.svg?logo=docker&logoColor=white)](https://docs.docker.com/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_workflow_building_blocks.md](01_workflow_building_blocks.md) | Workflow anatomy | Events, filters, jobs, expressions, outputs, artifacts, and environments |
| [02_pull_request_ci.md](02_pull_request_ci.md) | Pull-request CI | Secure validation and required-check design |
| [03_build_publish_and_promote.md](03_build_publish_and_promote.md) | Immutable builds | Publish once, identify by digest, and promote without rebuilding |
| [04_reusable_workflows_and_actions.md](04_reusable_workflows_and_actions.md) | Reuse | Reusable workflows, composite actions, and organization templates |
| [05_cross_repository_orchestration.md](05_cross_repository_orchestration.md) | Cross-repository delivery | Dispatch, GitOps, versioned dependencies, and synchronization |
| [06_performance_and_reliability.md](06_performance_and_reliability.md) | Pipeline operations | Matrices, caches, concurrency, timeouts, reruns, and cost |

---

## Reading Order

1. **Workflow building blocks** — learn the execution and data model
2. **Pull-request CI** — apply the model to safe validation

   **Stop here** if a required, secure PR check is all you need. Continue when you need to publish a deployable, immutable artifact.
3. **Build, publish, and promote** — produce deployable release candidates

   **Stop here** if you now have a digest-addressed artifact you can promote by hand. Continue when more than one repository needs to share this build/publish logic.
4. **Reusable workflows and actions** — standardize repeated logic

   **Stop here** if one repository owns the shared logic others call. Continue when your pipeline must trigger, wait on, or react to work in *other* repositories.
5. **Cross-repository orchestration** — coordinate independent repositories
6. **Performance and reliability** — tune the resulting system

---

## Prerequisites

- [CI/CD Foundations](../01_fundamentals/README.md)
- Basic YAML and shell knowledge
