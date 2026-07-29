# Cross-Repository Workflow Orchestration

> **Who this is for**: Teams coordinating application, platform, configuration, or dependent repositories. Read [Reusable Workflows and Actions](04_reusable_workflows_and_actions.md) first.

---

## 1. Start with the Relationship, Not the API

“Repo A checks Repo B” can mean several different architectures:

| Intent | Preferred pattern |
|--------|-------------------|
| Share the same CI/CD implementation | Cross-repository reusable workflow |
| Ask another repository to start a distinct process | `repository_dispatch` or `workflow_dispatch` |
| Release an application through a deployment repository | Versioned artifact plus GitOps PR |
| Update consumers of a library | Publish a version; dependency-update PRs |
| Distribute duplicated policy/configuration | Automated synchronization PR |
| Detect a source that cannot emit events | Scheduled polling |

```text
reuse       = same workflow run, shared implementation
dispatch    = a new asynchronous run in another repository
GitOps PR   = reviewed desired-state change
dependency  = versioned contract between repositories
sync        = duplicated files maintained by automation
polling     = consumer periodically compares upstream state
```

> **Key insight**: Prefer a versioned artifact or explicit contract over a dependency on “whatever is currently on another repository's main branch.”

---

## 2. Authentication across Repositories

The standard `GITHUB_TOKEN` is an installation token scoped to the repository containing the workflow. It is usually the right token inside that repository, but it cannot access arbitrary private repositories.

For cross-repository automation, prefer:

1. a GitHub App installation token restricted to selected repositories and permissions;
2. a fine-grained personal access token when an app is not practical;
3. a classic personal access token only for legacy compatibility.

Endpoint permission requirements differ:

| Operation | Fine-grained repository permission |
|-----------|-------------------------------------|
| Create `repository_dispatch` | Contents: write |
| Create `workflow_dispatch` | Actions: write |
| Open a pull request and push a branch | Contents: write and Pull requests: write |

Store a GitHub App's client ID as a variable and its private key as a secret. Installation tokens are short-lived.

---

## 3. Emit a Custom Repository Event

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
        uses: actions/create-github-app-token@v3
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
        uses: actions/create-github-app-token@v3
        with:
          client-id: ${{ vars.CICD_APP_CLIENT_ID }}
          private-key: ${{ secrets.CICD_APP_PRIVATE_KEY }}
          owner: acme
          repositories: orders

      - name: Verify artifact provenance
        env:
          GH_TOKEN: ${{ steps.source-token.outputs.token }}
          ARTIFACT: ${{ github.event.client_payload.image }}@${{ github.event.client_payload.digest }}
        run: gh attestation verify "oci://$ARTIFACT" --repo acme/orders
```

The payload is a request, not proof. Validate its schema, expected GitHub App actor, and allowlisted source, then verify the artifact independently. Replace the illustrative `acme-cicd[bot]` slug with the installed app's actual actor.

The receiving `repository_dispatch` workflow must exist on the destination's default branch.

---

## 4. Dispatch a Specific Workflow

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

## 5. Dispatch Is Asynchronous

A successful API response proves only that GitHub accepted the event.

```text
Repo A dispatch call ──202/204 or success──> event accepted
                                              │
                                              ▼
                                      Repo B run starts later
                                              │
                                    success / failure / timeout
```

Production orchestration needs correlation and completion semantics:

- include source run, commit, and artifact digest;
- capture the destination run URL when the API supplies one;
- report destination status back through a check, deployment status, or callback;
- impose a timeout and define what happens when the callback is lost;
- make duplicate events idempotent using the digest and target as a key.

Do not create an unbounded event loop:

```text
Repo A push → dispatch B → B push → dispatch A → ...
```

Use distinct event types, source allowlists, and an idempotency record.

---

## 6. Reusable Workflows across Repositories

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

No separate run starts in `platform-workflows`. Access settings must allow the caller to use private shared workflows, and the reference should be pinned.

---

## 7. GitOps Application and Deployment Repositories

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

The application repository owns building and testing. The deployment repository owns environment intent and promotion history. Production can require a PR while staging updates automatically.

Avoid storing cloud credentials in the application repository merely so it can modify production. The cross-repository GitHub App needs only enough access to open the desired-state PR.

---

## 8. Publish Versioned Dependencies

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

It is normally safer than copying the current source from Repo A.

---

## 9. Synchronize Files through Pull Requests

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
```

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

Event-driven delivery is usually faster and cheaper, but polling is a valid integration boundary for a producer you cannot change.

---

## 11. Decision Guide

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

---

**Next**: [Performance and Reliability](06_performance_and_reliability.md)
