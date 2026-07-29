# Security and Supply Chain

> Controls for workflow identity, untrusted code, runners, dependencies, and verifiable build outputs.

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-security-2088FF.svg?logo=githubactions&logoColor=white)](https://docs.github.com/actions)
[![Docker](https://img.shields.io/badge/Docker-supply_chain-2496ED.svg?logo=docker&logoColor=white)](https://docs.docker.com/)
[![AWS](https://img.shields.io/badge/AWS-OIDC-FF9900.svg?logo=amazonaws&logoColor=white)](https://docs.aws.amazon.com/)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_permissions_secrets_and_oidc.md](01_permissions_secrets_and_oidc.md) | Identity | Least-privilege tokens, secret boundaries, GitHub Apps, and cloud federation |
| [02_workflow_and_runner_hardening.md](02_workflow_and_runner_hardening.md) | Execution security | Action pinning, injection prevention, fork safety, and runner isolation |
| [03_sbom_provenance_and_attestations.md](03_sbom_provenance_and_attestations.md) | Supply-chain evidence | SBOMs, provenance attestations, signing, and verification |

---

## Reading Order

1. **Permissions, secrets, and OIDC** — limit who the workflow can become
2. **Workflow and runner hardening** — limit what workflow code can do
3. **SBOM, provenance, and attestations** — make outputs independently verifiable

---

## Prerequisites

- [GitHub Actions workflow building blocks](../02_github_actions/01_workflow_building_blocks.md)
- Familiarity with cloud IAM concepts
