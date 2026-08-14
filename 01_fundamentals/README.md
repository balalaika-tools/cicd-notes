# CI/CD Foundations

> The delivery model, repository controls, and test evidence that should exist before workflow YAML becomes complicated.

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-foundations-2088FF.svg?logo=githubactions&logoColor=white)](https://docs.github.com/actions)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_production_delivery_flow.md](01_production_delivery_flow.md) | Delivery flow | The path from a proposed change to a verified production release |
| [02_branching_and_change_control.md](02_branching_and_change_control.md) | Change control | Trunk-based development, rulesets, ownership, merge queues, and releases |
| [03_testing_and_quality_gates.md](03_testing_and_quality_gates.md) | Quality gates | Risk-based test layers and stable required checks |

---

## Reading Order

1. **Production delivery flow** — begin with the system and feedback loops

   **Milestone**: you can now explain the path from a proposed change to a verified release. Continue when you need to decide *which* changes are allowed to enter that path.
2. **Branching and change control** — control which changes may enter the system

   **Milestone**: you can now choose a branching model and the reviewer/ownership rules around it. **Stop here** if that's all your team needs today; continue when you need to define what evidence each merge gate must see before it lets a change through.
3. **Testing and quality gates** — define the evidence each change must produce

---

## Prerequisites

- Basic Git branching and pull-request knowledge
- Familiarity with automated tests
