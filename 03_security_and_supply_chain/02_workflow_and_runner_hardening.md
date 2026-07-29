# Workflow and Runner Hardening

> **Who this is for**: Engineers protecting CI/CD from malicious pull requests, compromised actions, secret exposure, and runner persistence. Read [Permissions, Secrets, and OIDC](01_permissions_secrets_and_oidc.md) first.

---

## 1. Model the Workflow as Privileged Code

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

The job is secure only when the input trust matches the runner, permissions, secrets, and network access.

---

## 2. Pin Actions to Full Commit SHAs

```yaml
# ✅ Immutable action source
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4
  with:
    persist-credentials: false

# ❌ A movable tag is easier to read but is not immutable
- uses: third-party/example-action@v2
```

Full-SHA pinning is the only action reference GitHub describes as immutable. Verify that the SHA belongs to the expected upstream repository, keep the release tag in a comment, and use Dependabot or Renovate to propose updates.

Apply the same rule to:

- reusable workflows from other repositories;
- composite actions;
- container actions and base images by digest;
- downloaded binaries and install scripts by checksum.

Repository and organization Actions policies can restrict which actions may run and require full-SHA pinning.

---

## 3. Prevent Script Injection

Vulnerable:

```yaml
- name: Check pull-request title
  run: |
    title="${{ github.event.pull_request.title }}"
    [[ "$title" =~ ^feat: ]]
```

GitHub expands the expression before the runner executes the generated shell script. A crafted title can break the assignment and inject commands.

Safer:

```yaml
- name: Check pull-request title
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    set -euo pipefail
    [[ "$PR_TITLE" =~ ^feat: ]]
```

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

## 4. Keep Privileged and Untrusted Work Apart

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

Avoid combining the two through `pull_request_target`.

If a privileged workflow consumes an artifact produced by an untrusted workflow:

- accept only a narrow data format;
- prevent path traversal during extraction;
- enforce size and file-count limits;
- verify checksums and expected producer identity;
- never execute scripts or binaries from the artifact;
- do not trust a filename or digest supplied only by the producer.

A `workflow_run` consumer can be privileged even when the producing workflow was not. The artifact boundary must be treated like an external upload.

---

## 5. Harden Checkout and Git Operations

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4
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

## 6. Minimize Action Trust

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

## 7. Use Ephemeral Self-Hosted Runners

GitHub-hosted runners provide a clean hosted environment per job. Self-hosted runners are appropriate for private networking, specialized hardware, compliance, or controlled toolchains, but the team owns isolation.

```text
job queued
    ↓
provision clean runner
    ↓
register as ephemeral / just-in-time
    ↓
execute one job
    ↓
forward logs externally
    ↓
destroy runner and storage
```

Production self-hosted controls:

- one job per ephemeral runner;
- immutable runner image and rapid patching;
- no public-repository or arbitrary-fork jobs on privileged runners;
- runner groups limited to approved repositories and workflows;
- minimal outbound network access and no broad internal-network reach;
- isolated cloud identity per workload;
- no long-lived credentials on disk;
- external logs because the runner is destroyed;
- resource and execution limits;
- cleanup verified after abnormal termination.

GitHub recommends ephemeral self-hosted runners for autoscaling. Actions Runner Controller is the recommended Kubernetes-based autoscaling solution for teams operating Kubernetes.

Persistent runners can leak workspace data, credentials, processes, Docker layers, or modified tooling into later jobs.

---

## 8. Protect Logs, Artifacts, and Caches

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

## 9. Protect Workflow Governance

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

## 10. Respond to a Workflow Compromise

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

Cache namespaces and runner images may need rotation even if no secret was visibly logged.

---

## 11. References

- [Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
- [Script injections](https://docs.github.com/en/actions/concepts/security/script-injections)
- [Self-hosted runners reference](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)
- [Managing Actions settings](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/managing-repository-settings/managing-github-actions-settings-for-a-repository)

---

**Next**: [SBOM, Provenance, and Artifact Attestations](03_sbom_provenance_and_attestations.md)
