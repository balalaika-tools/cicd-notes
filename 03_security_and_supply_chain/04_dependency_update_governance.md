# Dependency Update Governance

> **Who this is for**: Teams already running Dependabot or Renovate against SHA-pinned actions, digest-pinned base images, and pinned package versions, who now need a process for what happens after those bots open a PR.

## The short version

A repo that pins every action to a SHA and every base image to a digest still drifts if nobody handles what Dependabot or Renovate propose next: rubber-stamping every bot PR gives a first-time pin the same rubber stamp a routine patch bump gets, and ignoring the backlog leaves both routine bumps and the occasional vulnerable pin sitting open for months. The fix is a tiered gate — route each proposed update to auto-merge, mandatory human review, or an expedited security lane based on what actually changed, and require the exact same CI gate at every tier, so a bot-authored PR proves nothing less than a human-authored one would.

**What you need (4 things):**

1. An update-tool config (`dependabot.yml` or Renovate's `renovate.json`) that watches your ecosystems and groups routine bumps.
2. A workflow step that reads the proposed update's semver bump type and only *enables* auto-merge for the low-risk tier — it never merges outright.
3. A branch ruleset that requires the same status checks for every actor, with no bypass entry for the bot's own account.
4. A scheduled check (or the update tool's own alerting) that flags a pin whose age crosses a threshold with no update proposed.

**The code:**

```yaml
name: Dependabot auto-merge
on: pull_request
permissions:
  contents: write
  pull-requests: write
jobs:
  dependabot:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - id: metadata
        uses: dependabot/fetch-metadata@25dd0e34f4fe68f24cc83900b1fe3fe149efef98 # v3.1.0
        with:
          github-token: "${{ secrets.GITHUB_TOKEN }}"
      - name: Auto-merge the low-risk tier only
        if: >
          steps.metadata.outputs.update-type == 'version-update:semver-patch' ||
          steps.metadata.outputs.update-type == 'version-update:semver-minor'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: gh pr merge --auto --squash "${{ github.event.pull_request.html_url }}"
```
**Success signal:** a patch or minor Dependabot PR shows "This pull request will be merged automatically once required checks pass" in its merge box, then merges on its own the moment `required-ci` turns green — no reviewer opened the diff. A major-version PR gets no such message: the `if:` condition never matched, so it sits exactly where a human-authored major bump would.

**Not handled yet:** [routing major bumps, new sources, and first-time pins to mandatory review](#2-tier-the-pr-by-what-actually-changed-not-who-opened-it), [closing the bypass-list hole that lets a bot skip required checks](#3-the-required-ci-gate-has-no-bot-exception), [an expedited path for a CVE against a pin already in production](#4-a-cve-against-a-pinned-dependency-gets-a-clock-not-a-shortcut), and [catching a pin that just aged out with no CVE and no proposed update](#5-staleness-has-no-alert-by-default-so-you-have-to-build-one).

---

For the immutability argument behind the pin this note governs the lifecycle of, see [Only a Full Commit SHA Actually Pins an Action](02_workflow_and_runner_hardening.md#2-only-a-full-commit-sha-actually-pins-an-action) and [Unpinned Build Inputs Make Two Identical Builds Different](../02_github_actions/03_build_publish_and_promote.md#3-unpinned-build-inputs-make-two-identical-builds-different) — read either first if you haven't pinned anything yet. This note starts from "the pin already exists" and covers the PR that proposes moving it.

> **Core:** the auto-merge condition above, scoped to the low-risk tier, and the unconditional required-check ruleset in section 3 are the whole non-negotiable baseline. Sections 2, 4, and 5 are what you add once "patch bumps auto-merge" stops being the only case you have.

---

## 1. A Bot-Authored PR Is Either Rubber-Stamped or Ignored

`orders` pins every third-party action to a full SHA and every base image to a digest, exactly as the sections linked above recommend, and turns on Dependabot to keep those pins current. For the first month it works: a PR opens, a maintainer skims the diff, approves, merges. By month three, Dependabot has opened forty PRs against this one repository — patch bumps to `actions/checkout`, a minor bump to a logging library, one major bump that renamed a config key, and one PR pinning a brand-new action a teammate added to a workflow last week without ever running it past the trust checklist in [Workflow and Runner Hardening](02_workflow_and_runner_hardening.md#6-every-action-you-add-is-unreviewed-code-inside-your-job). Nobody has the appetite to review forty PRs a month with the care a human-authored change gets, so the team lands on one of two failure modes: approve everything on sight, including the unvetted new action, or let the backlog pile up unread, including the one PR that would have closed a real vulnerability. Either way, the pin — the thing supposedly protecting this repository — is only as trustworthy as whatever the team actually does with the PR proposing its next value.

The fix isn't more or less scrutiny across the board. It's noticing that not every PR proposing a new pin carries the same risk, and routing each one to the review it actually needs. The same three failure shapes apply to a base-image digest bump and to a package version pin — this note treats all three the same way, because the review question is identical: has whatever sits on the other end of this pin already earned the trust it's asking to keep.

---

## 2. Tier the PR by What Actually Changed, Not Who Opened It

> **Core:** three tiers are enough for almost every repository. Everything past the table below is how to detect which tier a given PR belongs to, and how to keep a grouped PR from quietly spanning two of them.

| Tier | Trigger signal | What happens |
|------|-----------------|--------------|
| **Auto-merge** | Patch or minor semver bump of a dependency already pinned by SHA, digest, or version, from a publisher you already trust | Auto-merge enabled once required checks pass; no human touches it |
| **Mandatory review** | Major version bump, **or** the pin's upstream source or publisher changed, **or** this is the first time this dependency is pinned at all | A human approves before merge; the same required checks still apply |
| **Expedited** | A security advisory names the SHA, digest, or version currently pinned | Section 4 — bounded time-to-merge, a narrower approver set, normal grouping suspended |

The first row's signal is a tool output, not a guess: `dependabot/fetch-metadata`'s `update-type` output reports exactly `version-update:semver-patch`, `-minor`, or `-major` for the dependency a PR proposes to bump, which is what the short version's `if:` condition reads. The other two rows in the mandatory-review tier aren't outputs at all — nothing in Dependabot's metadata flags "this dependency is new" or "this publisher changed." Detect both structurally instead: a first-time pin is a PR whose diff *adds* a `uses:`, `FROM`, or dependency-manifest line rather than only changing the trailing SHA, digest, or version on a line that was already there; a publisher change is a PR where the text to the left of the `@` changes — `foo/action` becomes `bar/action` after an action's ownership moves, or a base image's registry namespace changes. A `CODEOWNERS` entry on the manifest files themselves (`.github/workflows/`, `Dockerfile`, your lockfiles) routes a human onto exactly the PRs that touch dependency identity, not just its version — see [Branching and Production Change Control](../01_fundamentals/02_branching_and_change_control.md) for how that mapping already works for everything else in the repo.

> **Key insight**: risk tiering isn't about trusting the bot more or less — every PR here was opened by the same account. It's recognizing that "move this SHA forward" and "start trusting a new publisher" are different actions wearing an identical PR shape, and the signal that tells them apart is never the diff's size in bytes. It's whether the thing on the other end of the pin already earned the trust it's now asking to keep.

Grouping keeps the auto-merge tier from becoming forty separate PRs without blurring the tier boundary, as long as the group definition does the tier separation itself:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      actions-routine:
        applies-to: "version-updates"
        update-types: ["patch", "minor"]
    open-pull-requests-limit: 10

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      base-image-routine:
        applies-to: "version-updates"
        update-types: ["patch", "minor"]

  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      python-routine:
        applies-to: "version-updates"
        update-types: ["patch", "minor"]
```

Each `groups` block bundles same-tier updates into one PR — fewer PRs to review, still one tier per PR — by restricting `update-types` to `patch` and `minor`. A major bump is structurally excluded from every group above, so it always opens its own standalone PR, and that PR's `update-type` output never matches the auto-merge `if:` condition in the short version.

> **Edge case:** if a group's `update-types` list ever includes `"major"` alongside `"patch"`/`"minor"`, one bundled PR can carry a low-risk bump and a high-risk one under a single title and a single `update-type` verdict for the group. Treat any group definition that mixes `major` into an otherwise-routine group as un-auto-mergeable by policy — don't rely on the auto-merge job's `if:` condition to notice after the fact; keep `major` out of every group that feeds the auto-merge tier in the first place.

---

## 3. The Required-CI Gate Has No Bot Exception

Say a team, tired of Dependabot PRs blocking on a slow integration suite, adds `dependabot[bot]` to the repository ruleset's bypass list — GitHub's rulesets let you name an actor, including a bot, that skips the ruleset's protections entirely when it opens or merges a PR. An attacker who compromises an upstream package's publish pipeline — the same phished-token or hijacked-release-job attack [Workflow and Runner Hardening](02_workflow_and_runner_hardening.md#2-only-a-full-commit-sha-actually-pins-an-action) already walked through for actions — ships a malicious "patch" release. Dependabot's classifier sees a patch bump and proposes it exactly as designed. Because the bypass list skips required checks for this actor, the PR merges the moment it opens: no test run, no scan, nothing between the malicious patch and production.

```json
// ❌ Bypass list includes the update bot — its PRs skip required checks entirely
{
  "bypass_actors": [
    { "actor_id": 12345, "actor_type": "Integration", "bypass_mode": "always" }
  ]
}
```

```json
// ✅ No bypass entry for any bot — Dependabot's PRs answer to the same ruleset as a human's
{
  "bypass_actors": []
}
```

Point the ruleset's required-status-check setting at the same `required-ci` aggregator this collection already uses for human PRs — see [Harden the Required Check So a Skipped Job Cannot Pass It](../01_fundamentals/03_testing_and_quality_gates.md#3-harden-the-required-check-so-a-skipped-job-cannot-pass-it). A Dependabot PR satisfies that aggregator by making `lint`, `unit`, and every other upstream job actually run and succeed on its exact commit — not by the ruleset recognizing a bot actor and waiving the requirement. `gh pr merge --auto` in the short version's workflow only *enables* auto-merge; GitHub's own documentation is explicit that the option exists for PRs that "can't merge immediately" and defers the merge until required checks and reviews are satisfied — it is not a second path around the ruleset.

⚠️ The failure above leaves no error message, because nothing failed — the bypass list did exactly what it was configured to do. The tell is in the PR's own timeline, not in any command's exit code: open a merged Dependabot PR and check whether a `required-ci` check run exists on that commit at all. No entry there, on a PR that merged anyway, means required checks were bypassed or never configured on the branch — whether or not the auto-merge workflow "ran" is beside the point.

A repository already running a **merge queue** — GitHub's temporary environment where several merge candidates are re-validated together right before landing, described in [Approved Doesn't Mean Safe Until It's Tested Against Today's `main`](../01_fundamentals/02_branching_and_change_control.md#5-approved-doesnt-mean-safe-until-its-tested-against-todays-main) — doesn't need a different mechanism here: `gh pr merge --auto` adds the PR to that same queue instead of merging it directly, and the queue's own re-test against the latest `main` still runs before it lands.

---

## 4. A CVE Against a Pinned Dependency Gets a Clock, Not a Shortcut

None of the tiering in section 2 fires from a version-bump signal when the update exists because of a vulnerability, not a release — a semver classifier has no way to know a patch release closes a CVE, and it shouldn't have to. Dependabot security updates and Renovate's `vulnerabilityAlerts` are a separate trigger, matched against a security advisory rather than against a new upstream release, and they behave differently from routine version updates once they fire. GitHub's own docs state there is no interaction between `dependabot.yml` and Dependabot security alerts beyond one thing: merging the security PR closes the alert it addresses — so a security-update PR isn't shaped by your `schedule.interval` or your `open-pull-requests-limit` at all, and GitHub explicitly excludes security-update PRs from that limit. Renovate's equivalent is more direct about it: its docs describe `vulnerabilityAlerts` PRs as ones that "skip the line," ignoring `schedule`, `prConcurrentLimit`, and `prHourlyLimit` outright rather than reading an overridden value for any of them.

```json
{
  "vulnerabilityAlerts": {
    "labels": ["security"],
    "automerge": true,
    "assignees": ["@platform-oncall"]
  }
}
```

`automerge` here still means "once required checks pass," the same as section 3 — a security advisory changes the review tier and the clock, never whether `required-ci` has to be green. Three things change on this path relative to the routine one:

- **A bounded time-to-merge target.** A workable starting point, keyed to the advisory's severity: critical or high merges within 24–72 hours of the PR opening; medium within a week; low folds into the next routine batch instead of triggering the fast lane at all.
- **Who can approve outside the normal tier.** The security team or on-call responder can approve a fast-lane PR that would otherwise sit in the mandatory-review tier from section 2 — a major bump, say, or a first-time pin — because the cost of staying vulnerable now outweighs the cost of an unreviewed version bump. Required checks still gate the merge regardless of who clicks approve.
- **Normal grouping is suspended.** A CVE fix ships standalone, not folded into the week's grouped PR from section 2, so its audit trail and its rollback are independent of every unrelated bump riding alongside it.

> **Production:** enable Dependabot security updates (or Renovate's `vulnerabilityAlerts`) per repository before you need the fast lane, not while triaging the first advisory — the setting lives in the repository's own security configuration, not in `dependabot.yml`, and GitHub has moved that toggle's exact menu location before.

---

## 5. Staleness Has No Alert by Default, So You Have to Build One

A CVE gives a pin an alarm clock. Most pins never get one: an action's maintainers quietly stop cutting releases, a base image's upstream distro reaches end-of-life with no security bulletin attached, or `open-pull-requests-limit` from section 2 is already maxed out by PRs nobody merged, so Dependabot has stopped proposing anything new for that ecosystem. Every one of those looks identical from inside the repository — no new PR, no red banner, nothing to review — while the pin itself keeps getting older.

```yaml
name: Stale pin audit
on:
  schedule:
    - cron: "0 6 * * 1" # every Monday
  workflow_dispatch:

permissions:
  contents: read

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - name: Report action pins older than 90 days
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          set -euo pipefail
          max_age_days=90
          now_epoch=$(date -u +%s)
          grep -rhoE 'uses: [A-Za-z0-9._/-]+@[0-9a-f]{40}' .github/workflows/ | sort -u |
          while read -r _ target; do
            repo="${target%@*}"
            sha="${target#*@}"
            commit_date=$(gh api "repos/${repo}/commits/${sha}" --jq '.commit.committer.date')
            age_days=$(( (now_epoch - $(date -u -d "$commit_date" +%s)) / 86400 ))
            if (( age_days > max_age_days )); then
              echo "::warning::${repo}@${sha} is ${age_days} days old with no update proposed"
            fi
          done
```

`GH_TOKEN` here only needs to read public commit metadata on whatever repositories your `uses:` lines reference, which the default `GITHUB_TOKEN` already covers. The same pattern extends to base-image digests by walking `Dockerfile` lines instead of workflow files, and to package pins by comparing a lockfile's pinned version against the registry's latest release date for that package.

> **Production:** run this on a schedule, not as a manual audit — a threshold nobody checks is not a threshold. Route its warnings into whatever already tracks the CVE fast lane in section 4, so a stale pin becomes a ticket with an owner instead of a log line nobody reads. If you're already on Renovate, its optional `dependencyDashboard` config gives you a lower-effort version of this: one continuously updated issue listing every pending, rate-limited, or otherwise blocked update, instead of a script you maintain yourself.

---

## 6. What Goes Wrong in Practice, and When to Skip This Entirely

**A grouped PR quietly spans two tiers.** Section 2's edge case is the failure mode, not just a warning: a group whose `update-types` list was ever widened to include `major` produces a PR whose title says "Bump 6 actions" while one of the six is a major bump the auto-merge tier was never meant to see. The `update-type` output the auto-merge job reads describes the group's own classification, which is only as narrow as the group definition that produced it — there's no per-dependency check happening underneath.

⚠️ **A path filter on the CI workflow excludes the file Dependabot just changed.** This is the same drift [Testing and Quality Gates](../01_fundamentals/03_testing_and_quality_gates.md#3-harden-the-required-check-so-a-skipped-job-cannot-pass-it) already names for human-authored PRs, and it hits dependency PRs just as often: a workflow scoped to `paths: ['src/**']` never triggers on a PR that only touches `.github/workflows/ci.yml` or `go.mod`. `required-ci` never runs, the merge box shows it pending indefinitely, and the auto-merge job in the short version never gets a green check to act on. Nothing reads as broken; the PR just sits.

**`open-pull-requests-limit` is maxed out and nobody notices, because there's nothing to notice.** Settings shows the limit at its default of 5, and five Dependabot PRs have sat unmerged for months. The sixth update is never proposed until one of the five closes — the exact gap section 5's scheduled audit exists to catch, since Dependabot's own UI has no "I stopped proposing updates" banner.

Skip the tiering and auto-merge machinery entirely for a repository whose CI is a placeholder (`echo ok`) or nonexistent — a dependency PR can't be "proven safe by required checks passing" when nothing meaningful is actually exercised, and auto-merge there just removes the one review the change would otherwise get. It's also not worth building for a single-maintainer repository with a handful of dependencies, where a human already reads every diff faster than standing up this config would take. Reach for tiering once bot PR volume is high enough that consistent human review has already started slipping — the failure mode section 1 opened with.

---

## 7. References

- [Dependabot options reference](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference)
- [About Dependabot security updates](https://docs.github.com/en/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates)
- [Automating Dependabot with GitHub Actions](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/automating-dependabot-with-github-actions)
- [Creating rulesets for a repository](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository)
- [Managing auto-merge for pull requests](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-auto-merge-for-pull-requests-in-your-repository)
- [Renovate `vulnerabilityAlerts` configuration option](https://docs.renovatebot.com/configuration-options/#vulnerabilityalerts)
- [dependabot/fetch-metadata](https://github.com/dependabot/fetch-metadata)

---

**Next**: [Environments and Artifact Promotion](../04_delivery_operations/01_environments_and_promotions.md)
