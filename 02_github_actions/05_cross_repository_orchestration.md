# Cross-Repository Workflow Orchestration

<!-- length-justification: cross-repository orchestration is one decision space — relationship taxonomy, auth, dispatch, GitOps, versioning, sync, and polling all trade off against each other in the decision guide in §11, which needs every pattern in the same file to stay a useful single lookup. -->

> **Who this is for**: Teams coordinating application, platform, configuration, or dependent repositories.

## The short version

A workflow in Repo A can call a permitted reusable workflow stored in Repo B, but that workflow is incorporated into Repo A's run. Starting a distinct run owned by Repo B requires an event such as dispatch, a pull request, or a package release. Cross-repository orchestration means choosing which relationship you need and which credential, if any, may cross the repository boundary. The trace below shows the distinct-run dispatch case: an authenticated request from Repo A, accepted asynchronously by Repo B, and independently verified before Repo B acts on it.

**What you need (3 things):**

1. Two repositories with a defined relationship — Repo A initiates, Repo B receives.
2. A cross-repository credential — a GitHub App installation token scoped to Repo B (the default `GITHUB_TOKEN` can't reach outside its own repository).
3. A dispatch payload carrying the source repository, commit SHA, and artifact digest — the minimum Repo B needs to validate what it's being asked to act on.

**Worked example:**

```text
Repo A (acme/orders)                         Repo B (acme/platform-deployments)
─────────────────────                         ──────────────────────────────────
build succeeds on main
  │
  ├─ mint a Repo-B-scoped GitHub App token
  │
  └─ POST .../platform-deployments/dispatches
       event_type: release_candidate_published
       payload: source_sha=8f31c2a…, digest=sha256:4ae0…9c1d
                                               │
                                  202 accepted ┤
                                               ▼
                                  "Receive release candidate" run starts
```

**Success signal:** the `202` proves only event acceptance. Completion evidence is a new Repo B "Receive release candidate" run URL followed by its terminal result. If no run appears, check that the receiver exists on the default branch, its `types` includes the event type, and the App installation can access Repo B.

**Not handled yet:** [binding that artifact to the expected signer workflow and commit before trusting it](#3-a-dispatch-payload-is-a-claim-not-proof) and [tracking the destination run to a reported result instead of an accepted request](#5-accepted-is-not-completed-dispatch-needs-correlation-and-a-timeout).

---

For background on packaging a shared CI/CD implementation before you dispatch between repositories, see [Reusable Workflows and Actions](04_reusable_workflows_and_actions.md).

---

## 1. "Repo A Checks Repo B" Hides Six Different Relationships

"Repo A checks Repo B" can mean several different architectures:

| Intent | Preferred pattern |
|--------|-------------------|
| Share the same CI/CD implementation | Cross-repository reusable workflow |
| Ask another repository to start a distinct process | `repository_dispatch` or `workflow_dispatch` |
| Release an application through a deployment repository | Versioned artifact plus GitOps PR |
| Update consumers of a library | Publish a version; dependency-update PRs |
| Distribute duplicated policy/configuration | Automated synchronization PR |
| Detect a source that cannot emit events | Scheduled polling |
| Let an installed GitHub App or SaaS reconcile repository input | Externally managed integration with independent runtime verification |

```text
reuse       = same workflow run, shared implementation
dispatch    = a new asynchronous run in another repository
GitOps PR   = reviewed desired-state change
dependency  = versioned contract between repositories
sync        = duplicated files maintained by automation
polling     = consumer periodically compares upstream state
external    = repository input observed by an App/service-side control plane
```

For an external integration, a folder such as `deploy/orders/` is only source input. An App-authored check or comment proves detection; merge to the configured publication branch is the hand-off; the service then reconciles the mapped artifact into runtime. Record the administrator, connection name, watched repository/path, source-to-runtime identifier mapping, and runtime query. Treat watched branch and webhook-versus-polling behavior as `unknown` until the service administration page proves them. See [Reverse-Engineering External Repository Integrations](07_external_repository_integrations.md) for the full evidence procedure.

> **Key insight**: Prefer a versioned artifact or explicit contract over a dependency on "whatever is currently on another repository's main branch."

---

## 2. The Default `GITHUB_TOKEN` Stops at Your Own Repository

The standard `GITHUB_TOKEN` is an installation token scoped to the repository containing the workflow. It is usually the right token inside that repository, but it cannot access arbitrary private repositories.

For cross-repository automation, prefer:

1. a GitHub App installation token restricted to selected repositories and permissions;
2. a fine-grained personal access token when an app is not practical;
3. a classic personal access token only for legacy compatibility.

> **Core:** explicit cross-repository API or Git operations need destination authority broader than the source repository's default `GITHUB_TOKEN`. A permitted private reusable workflow is downloaded with a GitHub-provided scoped token, and polling a public producer needs no private-repository authority. For private reusable workflows, the central repository administrator must enable Actions access for the caller organization/repositories; a disallowed caller sees `workflow was not found`.

Endpoint permission requirements differ:

| Operation | Fine-grained repository permission |
|-----------|-------------------------------------|
| Create `repository_dispatch` | Contents: write |
| Create `workflow_dispatch` | Actions: write |
| Open a pull request and push a branch | Contents: write and Pull requests: write |

Store a GitHub App's client ID as a variable and its private key as a secret. Installation tokens are short-lived.

---

## 3. A Dispatch Payload Is a Claim, Not Proof

Repo A:

```yaml
name: Notify deployment repository

on:
  workflow_run:
    workflows: ["Build Release Candidate"]
    types: [completed]

permissions:
  actions: read
  contents: read

jobs:
  dispatch:
    if: >-
      ${{
        github.event.workflow_run.conclusion == 'success' &&
        github.event.workflow_run.head_branch == 'main'
      }}
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Download and validate the build's release manifest
        id: release
        env:
          GH_TOKEN: ${{ github.token }}
          SOURCE_RUN_ID: ${{ github.event.workflow_run.id }}
          EXPECTED_SHA: ${{ github.event.workflow_run.head_sha }}
        run: |
          set -euo pipefail
          mkdir release
          gh run download "$SOURCE_RUN_ID" \
            --name release-manifest \
            --dir release

          manifest=release/release-manifest.json
          source_sha="$(jq -er '.source.sha' "$manifest")"
          image="$(jq -er '.artifact.image' "$manifest")"
          digest="$(jq -er '.artifact.digest' "$manifest")"

          test "$source_sha" = "$EXPECTED_SHA"
          test "$image" = "ghcr.io/acme/orders"
          [[ "$digest" =~ ^sha256:[0-9a-f]{64}$ ]]

          echo "image=$image" >> "$GITHUB_OUTPUT"
          echo "digest=$digest" >> "$GITHUB_OUTPUT"

      # Pin this action to a reviewed full commit SHA in production.
      - name: Create a destination-scoped installation token
        id: app-token
        uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
        with:
          client-id: ${{ vars.CICD_APP_CLIENT_ID }}
          private-key: ${{ secrets.CICD_APP_PRIVATE_KEY }}
          owner: acme
          repositories: platform-deployments

      - name: Dispatch the immutable release identity
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          SOURCE_REPOSITORY: ${{ github.repository }}
          SOURCE_SHA: ${{ github.event.workflow_run.head_sha }}
          SOURCE_RUN_ID: ${{ github.event.workflow_run.id }}
          IMAGE: ${{ steps.release.outputs.image }}
          IMAGE_DIGEST: ${{ steps.release.outputs.digest }}
        run: |
          set -euo pipefail

          jq -n \
            --arg repository "$SOURCE_REPOSITORY" \
            --arg sha "$SOURCE_SHA" \
            --arg run_id "$SOURCE_RUN_ID" \
            --arg image "$IMAGE" \
            --arg digest "$IMAGE_DIGEST" \
            '{
              event_type: "release_candidate_published",
              client_payload: {
                source_repository: $repository,
                source_sha: $sha,
                source_run_id: $run_id,
                image: $image,
                digest: $digest
              }
            }' > dispatch.json

          gh api \
            --method POST \
            repos/acme/platform-deployments/dispatches \
            --input dispatch.json
```

The build publishes `release-manifest.json` as the `release-manifest` artifact. For long-lived promotion, also store or attest the manifest in a durable release store; the workflow artifact here transfers the immediate event data.

Repo B:

```yaml
name: Receive release candidate

on:
  repository_dispatch:
    types: [release_candidate_published]

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Validate the request envelope
        env:
          DISPATCH_ACTOR: ${{ github.actor }}
          SOURCE_REPOSITORY: ${{ github.event.client_payload.source_repository }}
          SOURCE_SHA: ${{ github.event.client_payload.source_sha }}
          IMAGE: ${{ github.event.client_payload.image }}
          DIGEST: ${{ github.event.client_payload.digest }}
        run: |
          set -euo pipefail
          test "$DISPATCH_ACTOR" = "acme-cicd[bot]"
          test "$SOURCE_REPOSITORY" = "acme/orders"
          test "$IMAGE" = "ghcr.io/acme/orders"
          [[ "$SOURCE_SHA" =~ ^[0-9a-f]{40}$ ]]
          [[ "$DIGEST" =~ ^sha256:[0-9a-f]{64}$ ]]

      # The destination GITHUB_TOKEN cannot read a different private repository.
      # Pin this action to a reviewed full commit SHA in production.
      - name: Create a source-repository verification token
        id: source-token
        uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
        with:
          client-id: ${{ vars.CICD_APP_CLIENT_ID }}
          private-key: ${{ secrets.CICD_APP_PRIVATE_KEY }}
          owner: acme
          repositories: orders
```

**A `--repo` check alone is not the same as verifying who built it.** `gh attestation verify "oci://$ARTIFACT" --repo acme/orders` only confirms the attestation's subject repository is `acme/orders` — it says nothing about which workflow ran, which branch it ran on, or which commit it built. Walk through what that leaves open:

1. `acme/orders` has more than one workflow that can call `actions/attest` — a nightly integration test, a docs-image build, anything with `id-token: write`. An attacker with write access to one of those (a merged PR to a low-review workflow, a compromised step in its build) pushes a throwaway branch that builds and attests an image there.
2. That workflow still runs inside `acme/orders`, using `acme/orders`'s own GitHub-issued OIDC identity, so its attestation is **completely valid** — Sigstore signs it, the transparency log records it, and `--repo acme/orders` matches.
3. The attacker gets that digest into the payload Repo B receives — by influencing whatever produces the dispatched digest, or by targeting a repository whose dispatch step trusts an easily-swayed input for it.
4. `gh attestation verify "oci://$ARTIFACT" --repo acme/orders` passes. The check only asked "did *some* workflow in `acme/orders` attest this digest?" — and the answer is yes.
5. Repo B promotes the image. The consumer believes they verified "the reviewed release build from `main`"; what they actually verified is "a workflow somewhere in `acme/orders` produced this digest, at some point, from some ref." That gap is the whole attack surface — nothing here required compromising `acme/platform-deployments` at all.

❌ `gh attestation verify "oci://$ARTIFACT" --repo acme/orders` — passes for any attested build from that repository.
✅ add `--signer-workflow`, `--source-ref`, and `--source-digest` — passes only for the release workflow, on `main`, building the exact commit the dispatch named.

```yaml
      - name: Verify artifact provenance
        env:
          GH_TOKEN: ${{ steps.source-token.outputs.token }}
          ARTIFACT: ${{ github.event.client_payload.image }}@${{ github.event.client_payload.digest }}
          SOURCE_SHA: ${{ github.event.client_payload.source_sha }}
        run: |
          gh attestation verify "oci://$ARTIFACT" \
            --repo acme/orders \
            --signer-workflow acme/orders/.github/workflows/release.yml \
            --source-ref refs/heads/main \
            --source-digest "$SOURCE_SHA"
```

`--repo` answers *which repository*. `--signer-workflow`, `--source-ref`, and `--source-digest` answer *which workflow, which branch, and which commit* — the same three facts the envelope check above already asserted from the untrusted payload, now checked against the signed certificate instead of the request that merely claims them. A build from the nightly test workflow, from any branch but `main`, or for any commit but the one the dispatch named fails verification even though its `acme/orders` attestation is genuine. (Source: [`gh attestation verify`](https://cli.github.com/manual/gh_attestation_verify), checked 2026-08-14.)

The payload is a request, not proof: validate the envelope's schema, actor, and source above, then verify the certificate's workflow, ref, and commit independently — one check is not a substitute for the other. Replace the illustrative `acme-cicd[bot]` slug with the installed app's actual actor.

The receiving `repository_dispatch` workflow must exist on the destination's default branch.

A `202` with no run means the hand-off failed after acceptance: the receiver may be absent from the default branch, `types` may not match `release_candidate_published`, or the App installation may not cover the destination. Only a destination run URL and status close that evidence gap.

---

## 4. `workflow_dispatch` Names the Job; `repository_dispatch` Names the Event

Use `workflow_dispatch` when Repo A knows exactly which destination workflow should run.

Repo B:

```yaml
name: Promote release

on:
  workflow_dispatch:
    inputs:
      source_repository:
        required: true
        type: string
      image:
        required: true
        type: string
      digest:
        required: true
        type: string
      target:
        required: true
        type: choice
        options:
          - staging
          - production

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - env:
          SOURCE_REPOSITORY: ${{ inputs.source_repository }}
          DIGEST: ${{ inputs.digest }}
        run: |
          set -euo pipefail
          test "$SOURCE_REPOSITORY" = "acme/orders"
          [[ "$DIGEST" =~ ^sha256:[0-9a-f]{64}$ ]]
```

Repo A, after creating a destination-scoped GitHub App token:

```yaml
- name: Start the destination workflow
  env:
    GH_TOKEN: ${{ steps.app-token.outputs.token }}
    IMAGE_DIGEST: ${{ needs.build.outputs.digest }}
  run: |
    set -euo pipefail
    gh workflow run promote.yml \
      --repo acme/platform-deployments \
      --ref main \
      -f source_repository="$GITHUB_REPOSITORY" \
      -f image="ghcr.io/acme/orders" \
      -f digest="$IMAGE_DIGEST" \
      -f target=staging
```

Use `workflow_dispatch` for an explicit command and `repository_dispatch` for a domain event that the receiver interprets.

---

## 5. Accepted Is Not Completed: Dispatch Needs Correlation and a Timeout

A successful API response proves only that GitHub accepted the event.

```text
Repo A dispatch call ──202/204 or success──> event accepted
                                              │
                                              ▼
                                      Repo B run starts later
                                              │
                                    success / failure / timeout
```

> **Production:** every dispatch-based integration needs correlation and completion semantics before it's trustworthy during an incident — the bullets below aren't optional polish on top of the mechanism.

Production orchestration needs correlation and completion semantics:

- include source run, commit, and artifact digest;
- capture the destination run URL when the API supplies one;
- report destination status back through a check, deployment status, or callback;
- impose a timeout and define what happens when the callback is lost;
- make duplicate events idempotent using the digest and target as a key.

The dispatch trace in [the receiver section above](#3-a-dispatch-payload-is-a-claim-not-proof) stops at "Repo B's run starts." Composing all five bullets into one flow looks like this:

```text
Repo A (acme/orders)                    Repo B (acme/platform-deployments)
─────────────────────                    ──────────────────────────────────
dispatch release_candidate_published
  correlation key: 8f31c2a…:staging
                                          "Receive release candidate" run starts
                                            │
                                            ├─ validate envelope, verify attestation (§3)
                                            │
                    commit status ◄─────────┤ report "pending", run URL attached
                    deploy/platform-deployments/staging = pending
                    (destination run now visible on Repo A's commit)
                                            │
                                            ├─ deploy job runs
                                            │
                    commit status ◄─────────┘ report terminal result, same run URL
                    deploy/platform-deployments/staging = success | failure
```

Repo B reports both ends of its own run back to Repo A, keyed by the same commit SHA and target it received in the payload — that pairing *is* the correlation ID. The callback is a small addition to the workflow in the section above:

```yaml
      - name: Report the destination run back to the source commit
        if: always()
        env:
          GH_TOKEN: ${{ steps.source-token.outputs.token }}
          SOURCE_SHA: ${{ github.event.client_payload.source_sha }}
          STATE: ${{ job.status == 'success' && 'success' || 'failure' }}
        run: |
          gh api \
            --method POST \
            repos/acme/orders/statuses/"$SOURCE_SHA" \
            -f state="$STATE" \
            -f context="deploy/platform-deployments/staging" \
            -f target_url="$GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID" \
            -f description="destination run $GITHUB_RUN_ID: $STATE"
```

**Success signal:** Repo A's commit shows a `deploy/platform-deployments/staging` status of `success`, with `target_url` pointing at Repo B's finished run — open it and the deploy logs are right there. Failure looks identical except for `failure`, with the same link, so a reader goes straight to the cause instead of re-deriving which run corresponds to which dispatch.

A lost callback — Repo B's runner is killed, the workflow is cancelled mid-job, the network drops before the last API call — never posts a terminal status at all, and without a timeout the commit sits at `pending` indefinitely, indistinguishable from "still deploying." Repo A needs its own watchdog, not a longer wait:

```text
Every 5 minutes, Repo A checks open correlations older than its timeout:
  deploy/platform-deployments/staging still "pending" at 8f31c2a…, age 24m (> 20m timeout)
    └─ POST repos/acme/orders/statuses/8f31c2a… state=error
         description: "no callback from platform-deployments within 20m — treat as failed"
```

⚠️ Without that watchdog, "pending" is not a status — it's silence. A `pending` commit status that is four hours old and one that is four seconds old look identical to anyone reading the check; only a timeout that actively writes `error` after the deadline turns a hang into something the reader can actually see and act on.

Do not create an unbounded event loop:

```text
Repo A push → dispatch B → B push → dispatch A → ...
```

> **Edge case:** two repositories dispatching to each other on every push is the runaway failure mode of this whole pattern family. Use distinct event types, source allowlists, and an idempotency record — the same digest-plus-target key used for correlation above — so a legitimate retry can't become a loop.

---

## 6. A Cross-Repository Reusable Workflow Is Still One Run, Not Two

Use this when the goal is shared implementation:

```yaml
jobs:
  ci:
    uses: acme/platform-workflows/.github/workflows/python-ci.yml@0123456789abcdef0123456789abcdef01234567
    with:
      python_version: "3.12"
```

```text
caller repository
    └── one workflow run
         └── jobs defined by central reusable workflow
```

No separate run starts in `platform-workflows`. Access settings must allow the caller to use private shared workflows.

> **Production:** pin the reference to a reviewed full commit SHA, as shown above, before it reaches a real deploy path. A movable tag or branch ref lets `platform-workflows` change your CI without your review — the exact risk the pinned SHA in [§3](#3-a-dispatch-payload-is-a-claim-not-proof) and [§4](#4-workflow_dispatch-names-the-job-repository_dispatch-names-the-event) is guarding against for actions.

---

## 7. The App Repo Builds; the Deployment Repo Owns What's Live

```text
application repository
├── source and tests
├── build image once
├── publish digest + provenance
└── open deployment-repo PR
              │
              ▼
deployment repository
├── environment manifests
├── policy and ownership review
├── change image digest
└── merge desired state
              │
              ▼
Argo CD / Flux / deployment workflow
├── reconcile target
├── report health
└── update deployment status
```

Example desired-state change:

```yaml
service: orders
image:
  repository: ghcr.io/acme/orders
  digest: sha256:4ae0...9c1d
source:
  repository: acme/orders
  commit: 8f31c2a7d9...
```

The application repository owns building and testing. The deployment repository owns environment intent and promotion history.

> **Production:** require a PR and review for production manifests even if staging auto-merges. That split — reviewed change for production, fast path for staging — is what makes this a GitOps repository instead of an unreviewed direct push to what's live.

Avoid storing cloud credentials in the application repository merely so it can modify production. The cross-repository GitHub App needs only enough access to open the desired-state PR.

---

## 8. A Version Number Is a Safer Contract Than "Whatever's on Main"

For libraries, schemas, actions, Terraform modules, and Helm charts:

```text
Repo A publishes version 2.4.1
        │
        ▼
package or artifact registry
        │
        ▼
Dependabot/Renovate opens Repo B PR
        │
        ▼
Repo B tests compatibility before merge
```

This makes the relationship explicit:

```toml
shared-contract = "2.4.1"
```

The complete local carrier includes all five boundaries:

```text
producer metadata:  name=shared-contract version=2.4.1 schema_digest=sha256:91ab...
registry result:    shared-contract@2.4.1 published, immutable digest sha256:91ab...
consumer lock:      shared-contract = "2.4.1"; integrity = "sha256:91ab..."
updater PR:         2.4.1 → 2.5.0; producer/consumer compatibility suite = PASS
deprecation:        2.x supported through 2027-03-31; removal requires 3.0.0
```

The producer owns publication and backward-compatibility policy; the consumer owns adoption. Without registry publication evidence and the updater's effective configuration, this is an architectural proposal, not proof of an existing binding. [Cross-Repository Contract Lifecycle](08_cross_repository_contract_lifecycle.md) owns the complete publish-to-deprecation walkthrough.

It is normally safer than copying the current source from Repo A.

---

## 9. Copies Drift — Sync Through a Reviewed PR, Not a Force-Push

Sometimes organization-owned files must exist in every repository:

- `CODEOWNERS`;
- Dependabot configuration;
- lint or policy configuration;
- issue templates;
- legal notices;
- small workflow callers.

Use a bot-created PR rather than force-pushing destination `main`:

```text
central policy repository
    ↓ render target-specific files
    ↓ validate destination diff
    ↓ open or update a deterministic bot branch
destination pull request
    ↓ normal review and required checks
    ↓ merge to configured publication branch
destination/runtime reports consumed rendered digest
```

Write down the synchronization contract: `acme/policy` is canonical; `templates/dependabot.yml` renders `.github/dependabot.yml`; the bot watches `main` and that path; `acme-policy-sync[bot]` opens or updates branch `automation/policy-sync`; required checks validate compatibility; after merge, the destination reports the rendered checksum it loaded. A detected diff proves only that the bot noticed a mismatch. The merged checksum and independent destination/runtime observation prove consumption.

Prefer reusable workflows and versioned packages where possible. Synchronization creates copies, and copies drift.

---

## 10. Poll Only When the Producer Cannot Notify

A scheduled consumer can compare:

- upstream release version;
- package version;
- manifest checksum;
- commit SHA;
- artifact digest.

Store the last consumed immutable identifier, not only the poll time. Polling must tolerate missed schedules, rate limits, and the same version being observed more than once.

> **Edge case:** reach for polling only when you cannot modify the producer at all. Every other pattern in this note assumes you can add a dispatch, a callback, or a published version — polling is what's left when you can't.

Event-driven delivery is usually faster and cheaper, but polling is a valid integration boundary for a producer you cannot change.

---

## 11. Pick the Pattern From the Requirement, Not the Habit

```text
Need shared jobs and policy?
└── reusable workflow

Need a separate destination process?
├── domain event → repository_dispatch
└── exact workflow → workflow_dispatch

Need environment desired state and an approval history?
└── GitOps deployment repository

Need consumer compatibility?
└── versioned package + automated update PR

Need identical repository-owned files?
└── synchronization PR

Cannot modify producer?
└── scheduled polling
```

---

## 12. References

- [Triggering a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow)
- [Create a repository dispatch event](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event)
- [Create a workflow dispatch event](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event)
- [Making authenticated API requests with a GitHub App](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/making-authenticated-api-requests-with-a-github-app-in-a-github-actions-workflow)
- [Sharing private actions and workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/share-across-private-repositories)
- [`gh attestation verify` reference](https://cli.github.com/manual/gh_attestation_verify)

---

**Next**: [Reverse-Engineering External Repository Integrations](07_external_repository_integrations.md)
