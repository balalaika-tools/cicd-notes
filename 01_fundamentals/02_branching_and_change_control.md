# Branching and Production Change Control

> **Who this is for**: Teams deciding how source changes become authorized release candidates. Read [Production CI/CD as a Delivery System](01_production_delivery_flow.md) first.

---

## 1. Prefer a Deployable Main Line

For most product teams, a strong default is **trunk-based development**:

```text
main ──●────────●────────●────────●────────────>
        \      /          \      /
         ●──●─●            ●──●─●
       short-lived        short-lived
       feature branch     feature branch
```

- Branches live for hours or a few days, not weeks.
- Pull requests are small and merged frequently.
- `main` remains releasable.
- Incomplete behavior is hidden behind feature flags or dark launches.
- Release metadata identifies production versions; environment branches do not.

Long-lived `develop`, `release/*`, or environment branches can be appropriate when supporting multiple released product lines or constrained release trains. They also create merge debt and make it harder to know which branch contains production truth.

| Need | Usually choose |
|------|----------------|
| SaaS with frequent releases | Trunk-based development |
| Mobile or embedded versions supported in parallel | Release branches plus a main line |
| Open-source library with maintained majors | Maintenance branches and version tags |
| Environment configuration separated from code | GitOps configuration repository, not app environment branches |

---

## 2. Protect the Main Line with Rulesets

GitHub rulesets or branch protection should enforce the repository contract:

```text
main
├── pull request required
├── 1–2 approving reviews
├── code-owner review for sensitive paths
├── stale approvals dismissed after material changes
├── required status checks
├── conversation resolution required
├── merge queue for busy repositories
├── force pushes and deletion blocked
└── bypass restricted and audited
```

Rulesets can target branches and tags and can be layered at repository and organization level. Keep bypass lists short. An emergency path should relax time or reviewer requirements without bypassing build integrity, audit logging, or post-deployment verification.

> **Rule**: Administrators should be subject to the normal path unless they are executing a documented, audited exception.

Stable required checks matter. If a check name changes with a matrix value or a path-filtered job never reports, GitHub may not receive the status that the ruleset expects. Use a final, consistently named gate job as described in [Testing and Quality Gates](03_testing_and_quality_gates.md).

---

## 3. Express Ownership with `CODEOWNERS`

`CODEOWNERS` requests the reviewers who understand sensitive areas.

```text
# .github/CODEOWNERS

# Default ownership
*                         @acme/application-team

# Delivery and security boundaries
/.github/workflows/       @acme/platform-team @acme/security-team
/infra/                   @acme/platform-team
/migrations/              @acme/database-team
/security/                @acme/security-team

# High-risk business logic
/src/payments/            @acme/payments-team
```

Combine the file with a ruleset requiring code-owner approval. Merely requesting a code owner does not ensure approval.

Ownership rules should reflect real response capacity. A team that no longer monitors reviews becomes a release bottleneck and encourages bypasses.

---

## 4. Design Pull Requests for Evidence

A production-friendly change is:

- small enough to review carefully;
- independently testable;
- backward-compatible during rollout;
- observable after deployment;
- reversible or protected by a feature flag;
- accompanied by a migration and recovery plan when state changes.

A useful pull-request template asks for decisions rather than ceremony:

```markdown
## Intent

What behavior or operational property changes?

## Evidence

- Tests:
- Security or data considerations:
- Screenshots, plans, or benchmark:

## Delivery

- Rollout strategy:
- Signals to watch:
- Rollback or roll-forward action:
- Feature flag:
```

Avoid asking authors to duplicate information that automation can attach.

---

## 5. Use Merge Queues When Integration Moves Quickly

A pull request can pass against yesterday's `main` and fail after another pull request merges. A merge queue tests the candidate against a current integration branch before merging.

```text
approved PR A ─┐
approved PR B ─┼─> temporary merge group ─> required checks ─> main
approved PR C ─┘
```

GitHub Actions workflows that supply required merge-queue checks must listen for `merge_group`:

```yaml
name: Required CI

on:
  pull_request:
    branches: [main]
  merge_group:
    types: [checks_requested]

permissions:
  contents: read

jobs:
  required-ci:
    name: required-ci
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4
      - run: ./scripts/ci.sh
```

⚠️ Adding a merge queue without the `merge_group` trigger can leave queued changes without the checks needed to merge.

---

## 6. Choose Release Triggers Deliberately

| Trigger | Strength | Risk |
|---------|----------|------|
| Push to `main` | Every accepted change is a candidate | Requires excellent automated gates |
| Signed or protected version tag | Explicit release identity | Tags can point to untested commits if not controlled |
| GitHub Release | Human-visible release event | Do not rebuild different content during publication |
| Manual `workflow_dispatch` | Useful for recovery and controlled promotion | Caller can select incorrect inputs unless validated |
| Change in GitOps repository | Deployment intent is reviewed separately | Requires reconciliation and cross-repository observability |

A safe tag flow creates a tag from an already-built, tested commit and maps it to the existing digest:

```text
commit 8f31c2a ──build──> digest sha256:4ae0...
       │
       └── tag v2.7.0 ──release metadata──> same digest
```

Do not rebuild the tag and assume its output matches the main-branch build.

---

## 7. Emergency Change Path

Define the exception before an incident:

```text
incident commander authorizes emergency
    ↓
small PR with linked incident
    ↓
mandatory build and security checks
    ↓
reduced but independent review
    ↓
production environment + deployment record
    ↓
heightened verification
    ↓
follow-up review and corrective work
```

The emergency path may shorten waiting periods. It should not permit unreviewed workflow modification and privileged deployment in the same change.

---

## 8. Common Failure Modes

**Required checks can be skipped**

Path filters or conditional jobs never create the expected check. Use a stable aggregator job that always runs.

**A compromised contributor changes the pipeline and the product together**

Require platform or security ownership for `.github/workflows/**`, runner configuration, and deployment code.

**Feature branches diverge for weeks**

Split work behind flags, merge compatible slices, and avoid branch-based integration as a substitute for architecture.

**Release tags bypass main-line evidence**

Restrict tag creation, verify the tagged commit, and promote the artifact already associated with that commit.

---

## 9. References

- [Available rules for GitHub rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets)
- [About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [Managing a merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)

---

**Next**: [Testing and Quality Gates](03_testing_and_quality_gates.md)
