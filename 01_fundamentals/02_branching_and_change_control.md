# Branching and Production Change Control

> **Who this is for**: Teams deciding how source changes become authorized **release candidates** — specific built artifacts eligible for promotion after the required evidence passes.

## The short version

A pull request can be approved and still merge code nobody safely tested — the approval might be against yesterday's `main`, or from someone who doesn't own the path being changed.
**Change control** is the repository configuration that closes that gap: chiefly a GitHub **ruleset** (a named branch/tag protection policy GitHub enforces on its own server, not something a contributor's local git client can skip) that makes review, ownership, and a fresh test run mandatory rather than a matter of discipline.
Four concrete inputs are enough to make that guarantee hold for every change, including ones from people with admin access.

**What you need (4 things):**

1. A branch ruleset on `main` requiring a pull request, an approving review, and a status check.
2. A `CODEOWNERS` file (a repo-root file mapping paths to the reviewers required for them) mapping the changed path to its owning team.
3. A required-check workflow that reports one stable, consistently named result.
4. A short-lived branch carrying the actual change.

**Worked example:** Mia branches `feature/payments-retry` off `main` and edits `/src/payments/retry.py`. `CODEOWNERS` maps that path to the payments team:

```text
# .github/CODEOWNERS

# Default ownership — anyone can approve routine changes
*                         @acme/application-team

# Sensitive paths need their owning team's approval too
/.github/workflows/       @acme/platform-team @acme/security-team
/infra/                   @acme/platform-team
/migrations/              @acme/database-team
/src/payments/            @acme/payments-team
```

When she opens the pull request, GitHub requests review from `@acme/payments-team` and disables the merge button until a member of that team approves and `required-ci` reports success on her exact commit.
An approval from `@acme/application-team`, or a passing check on an earlier commit, satisfies neither requirement.

The effective server-side rule must also be inspected. This abbreviated exported ruleset is a concrete carrier, not a file the repository itself owns:

```json
{
  "name": "main-change-control",
  "enforcement": "active",
  "target": "branch",
  "conditions": {"ref_name": {"include": ["~DEFAULT_BRANCH"]}},
  "bypass_actors": [],
  "rules": [
    {"type": "pull_request", "parameters": {"required_approving_review_count": 1, "require_code_owner_review": true}},
    {"type": "required_status_checks", "parameters": {"required_status_checks": [{"context": "required-ci", "integration_id": 15368}]}}
  ]
}
```

The repository proves the `CODEOWNERS` mapping and workflow definition. A repository or organization administrator must query every active repository-, organization-, and enterprise-level ruleset targeting `main` and confirm enforcement mode, bypass actors, review count, code-owner review, and both the check name and expected source App. Without that access those facts remain `unknown`. The negative proof is a PR that changes `/src/payments/`, has only a non-owner approval or a same-named status from another source, and remains blocked.

**Success signal:** The merge box reads "Review required from @acme/payments-team," and the merge button stays disabled until that review lands and `required-ci` is green on the current commit.

**Not handled yet:** [what happens once several approved PRs queue together](#5-approved-doesnt-mean-safe-until-its-tested-against-todays-main), [the path for changes that can't wait for normal review](#7-define-the-emergency-path-before-the-incident-not-during-it), [which release trigger to use once the PR merges](#6-choose-the-release-trigger-by-what-evidence-it-preserves).

---

For the pipeline stages a change moves through after it merges, see
[Production CI/CD as a Delivery System](01_production_delivery_flow.md). Not needed for
the baseline above.

## 1. Long-Lived Branches Create Merge Debt — Keep `main` Releasable Instead

Two engineers each branch off `main` and work for three weeks: one refactors the payments
module, the other adds a field to the same models. Neither integrates until "feature
complete." When they finally open pull requests, each diff conflicts with dozens of
commits that landed on `main` since they branched, and resolving those conflicts by hand
silently reintroduces bugs each branch's own tests already caught once. That accumulated
resolution cost is **merge debt** — work created by divergence itself, not by either
feature — and it compounds faster than the actual scope of either change.

For most product teams, the fix is **trunk-based development**: branches live for hours
or days, not weeks, so a diff is too small and too fresh to conflict with much.

```text
main ──●────────●────────●────────●────────────>
        \      /          \      /
         ●──●─●            ●──●─●
       short-lived        short-lived
       feature branch     feature branch
```

> **Core:** short-lived branches and a releasable `main` are the baseline every team
> needs. Long-lived branching is a deliberate, scoped exception — not a fallback for
> teams that haven't gotten around to trunk-based development.

- Branches live for hours or a few days, not weeks.
- Pull requests are small and merged frequently.
- `main` remains releasable.
- Incomplete behavior stays hidden behind feature flags or a **dark launch** — code
  deployed to production but not yet exposed to real users or traffic.
- Release metadata identifies production versions; environment branches do not.

Long-lived `develop`, `release/*`, or environment branches can be appropriate when
supporting multiple released product lines or constrained release trains. Choose that
deliberately: it buys real flexibility, but it reintroduces the merge debt above.

| Need | Usually choose |
|------|----------------|
| SaaS with frequent releases | Trunk-based development |
| Mobile or embedded versions supported in parallel | Release branches plus a main line |
| Open-source library with maintained majors | Maintenance branches and version tags |
| Environment configuration separated from code | **GitOps** (a reconciled desired-state approach: a controller continuously applies what a Git repository declares) configuration repository, not app environment branches |

---

## 2. A Ruleset Beats an Honor System — Including for Admins

An administrator (or anyone GitHub grants bypass rights) can push straight to `main`,
skipping pull request review entirely, because "administrator" usually means "not subject
to the rules everyone else follows." Suppose that person pushes one commit that both
modifies `.github/workflows/deploy.yml` — adding a step — and changes `/src/payments/`.
With no review gate, that combined change reaches `main`, and the next pipeline run
executes the modified workflow, with whatever repository secrets and deployment
permissions that workflow already carries, before anyone else has seen the diff. The
ruleset below closes that gap for everyone with write access, including admins, by
turning one thing into an explicit, logged decision: a bypass is a documented exception
invoked for a specific change, not a standing privilege that comes with the role.

GitHub rulesets should enforce the repository contract. ★ marks the baseline most
repositories need on day one; the rest are conditional additions layered on once that
baseline is in place:

```text
main
├── ★ pull request required
├── ★ 1–2 approving reviews
├── ★ required status checks
├── ★ force pushes and deletion blocked
├── ★ bypass restricted and audited
├── code-owner review for sensitive paths       # needs CODEOWNERS, see §3
├── stale approvals dismissed after material changes
├── conversation resolution required
└── merge queue for busy repositories            # batches approved PRs through a
                                                  # temporary integration branch — see §5
```

Rulesets can target branches and tags and can be layered at repository and organization
level. Keep bypass lists short. An emergency path should relax time or reviewer
requirements without bypassing build integrity, audit logging, or post-deployment
verification — see [§7](#7-define-the-emergency-path-before-the-incident-not-during-it).

> **Rule**: Administrators should be subject to the normal path unless they are executing
> a documented, audited exception. The ruleset's job is to keep "who can bypass, and for
> what" a visible, logged decision — not an unwritten property of who holds admin access.

Stable required checks matter, and the mechanism behind that requirement isn't obvious
until you've watched it fail: if a required check's name changes because it runs inside a
**matrix** (one job template GitHub Actions runs repeatedly with different input values —
say, once per OS or language version — each producing its own differently named check),
or a path-filtered job never triggers for this diff, GitHub is waiting for a status report
that will never arrive. The pull request sits un-mergeable with no error message, only a
permanently pending check. Use a final, consistently named gate job as described in
[Testing and Quality Gates](03_testing_and_quality_gates.md).

---

## 3. Split Ownership So One Compromised Account Can't Authorize Everything

A contributor's laptop is compromised, or their token leaks. The attacker opens a pull
request from that account changing both `.github/workflows/ci.yml` — adding a step that
exfiltrates secrets — and `/src/payments/` — redirecting a slice of transactions. If the
only review requirement is "one approval from anyone with write access," a single rushed
or equally compromised reviewer can wave through both changes as one diff, because
nothing forces the workflow change past the team that actually understands what that
workflow is trusted to do. `CODEOWNERS` closes that gap by requiring approval from the
team that owns *each* sensitive path, so authorizing the payments change and authorizing
the workflow change become two separately exercised approvals — no single compromised
account or reviewer can grant both at once.

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

Combine the file with a ruleset requiring code-owner approval (the conditional entry in
the §2 tree). Merely requesting a code owner does not ensure approval, and the two states
look almost identical in the pull request sidebar.

**How you know it's working:** open a test pull request that touches only
`/src/payments/`. The sidebar should show "Review required from @acme/payments-team," and
the merge button should stay disabled until that team approves — even if someone from
`@acme/application-team` already approved. If the sidebar requests the payments team but
the merge button is still clickable without their approval, code-owner approval is
*requested* but not *enforced*: the ruleset's "require review from Code Owners" setting
isn't on, and `CODEOWNERS` is doing nothing but populating a reviewer suggestion.

Ownership rules should reflect real response capacity. A team that no longer monitors
reviews becomes a release bottleneck and encourages bypasses.

---

## 4. Ask the Pull Request for Evidence, Not Ceremony

A production-friendly change is:

- small enough to review carefully;
- independently testable;
- backward-compatible during rollout;
- observable after deployment;
- reversible or protected by a feature flag;
- accompanied by a migration and recovery plan when state changes.

> **Core:** these six properties matter more than any template wording — the template
> below just prompts an author for evidence that they're actually true.

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

## 5. Approved Doesn't Mean Safe Until It's Tested Against Today's `main`

A pull request can pass its checks against `main` as it stood yesterday, and fail the
moment another pull request merges ahead of it — two changes that are individually fine
can still misbehave once combined. A **merge queue** serializes this: instead of merging
an approved PR straight into `main`, GitHub creates a temporary integration branch
combining it with `main` plus any other approved PRs ahead of it, re-runs required checks
against that combination, and only then updates `main` using the configured merge, rebase,
or squash method. That method is external queue configuration, not workflow-owned behavior.

```text
approved PR A ─┐
approved PR B ─┼─> temporary merge group ─> required checks ─> main
approved PR C ─┘
```

The mechanism that trips people up: GitHub represents each queued candidate with its own
**merge-group ref** (something like `refs/heads/gh-readonly-queue/main/pr-42-...`), not a
pull request. A workflow that only triggers `on: pull_request` never fires for that ref —
so the very check your ruleset requires simply never runs for the queued candidate, and
the queue has nothing to advance on.

GitHub Actions workflows that supply required merge-queue checks must also listen for
`merge_group`:

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
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - run: ./scripts/ci.sh
```

⚠️ Skip the `merge_group` trigger and the symptom is exactly what the mechanism above
predicts: the queued candidate gets no `required-ci` check at all — not a failing one, an
absent one — and it stalls with no error pointing at the cause.

> **Production:** pin `actions/checkout` to a commit SHA you've reviewed, not a moving
> tag — a compromised or force-pushed tag would otherwise change what your pipeline runs
> without changing your workflow file. `v7.0.1` is the [current release](https://github.com/actions/checkout/releases)
> as of this note's last check; treat any fixed SHA as something you revisit; GitHub
> ships [security backports](https://github.blog/changelog/2026-06-18-safer-pull_request_target-defaults-for-github-actions-checkout/)
> only to release lines still receiving updates, so a stale pin doesn't get them
> automatically.

**How you know it's working:** queue a test pull request — or let it enter automatically
once approved and its checks pass — then watch the Checks tab for a *new* run whose event
is `merge_group`, with a job named `required-ci`. GitHub only advances the candidate into
`main` after that run reports success. If the merge group shows no `required-ci` check at
all, not even pending, the workflow isn't listening for `merge_group`, and the PR will sit
queued until it times out or someone dequeues it manually.

---

## 6. Choose the Release Trigger by What Evidence It Preserves

| Trigger | Strength | Risk |
|---------|----------|------|
| Push to `main` | Every accepted change is a candidate | Requires excellent automated gates |
| Signed or protected version tag | Explicit release identity | Tags can point to untested commits if not controlled |
| GitHub Release | Human-visible release event | Do not rebuild different content during publication |
| Manual `workflow_dispatch` | Useful for recovery and controlled promotion | Caller can select incorrect inputs unless validated |
| Change in GitOps repository | Deployment intent is reviewed separately | Requires reconciliation and cross-repository observability |

A safe tag flow creates a tag from an already-built, tested commit and maps it to the
existing digest:

```text
commit 8f31c2a ──build──> digest sha256:4ae0...
       │
       └── tag v2.7.0 ──release metadata──> same digest
```

> **Edge case:** rebuilding from the tag instead of promoting the artifact already built
> from that commit is the most common way a "signed release" silently stops matching what
> was actually tested — the tag looks authoritative, but the bytes behind it never ran
> through the gates the ruleset enforced.

---

## 7. Define the Emergency Path Before the Incident, Not During It

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

The emergency path may shorten waiting periods. It should not permit unreviewed workflow
modification and privileged deployment in the same change.

> **Edge case:** invoke this path only during a declared incident with a named
> commander. Using it as a faster default review process defeats the reason the ruleset
> in §2 exists at all.

---

## 8. What Still Breaks Even With the Ruleset Configured Correctly

**Required checks can be skipped**

Path filters or conditional jobs never create the expected check. Use a stable aggregator
job that always runs — the same failure mode explained in §2.

**A compromised or careless account tries to bundle a workflow change with a product
change**

See [§3](#3-split-ownership-so-one-compromised-account-cant-authorize-everything) for why
independent code-owner approval keeps one approval from covering both.

**Feature branches diverge for weeks**

Split work behind flags, merge compatible slices, and avoid branch-based integration as a
substitute for architecture.

**Release tags bypass main-line evidence**

Restrict tag creation, verify the tagged commit, and promote the artifact already
associated with that commit.

> **Key insight**: none of this strength comes from the number of branches, rules, or
> approvals configured. It comes from requiring evidence that was produced independently
> of the person or account proposing the change, and from making every way around that
> requirement a narrow, logged exception instead of a standing privilege.

---

## 9. References

- [Available rules for GitHub rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets)
- [About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [Managing a merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)

---

**Next**: [Testing and Quality Gates](03_testing_and_quality_gates.md)
