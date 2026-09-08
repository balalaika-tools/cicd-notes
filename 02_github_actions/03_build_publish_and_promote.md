# Build, Publish, and Promote

> **Who this is for**: Teams turning an accepted commit into an immutable release candidate.

## The short version

A rebuild between staging and production can silently ship different bytes than the ones that were tested — a moved base-image tag, a re-resolved dependency, or a different compiler patch, even from identical source. The fix is to build the release artifact exactly once, publish it under a content digest, and promote that digest — never rebuild it — into every later environment. A digest is sufficient identity because it is a hash of the image's actual bytes: two builds share a digest only if every byte matches.

**What you need (3 things):**

1. A build job that publishes an image to a registry and exposes its digest as a job output.
2. A digest-qualified reference (`image@sha256:...`), never a moving tag, passed between jobs.
3. A deploy step that accepts that image and digest and applies them without rebuilding.

**The code:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ github.token }}
      - id: build
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/acme/orders:git-${{ github.sha }}
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Apply the digest and wait for it to serve
        env:
          DIGEST: ${{ needs.build.outputs.digest }}
        run: |
          set -euo pipefail
          kubectl set image deployment/orders orders="ghcr.io/acme/orders@${DIGEST}"
          kubectl rollout status deployment/orders --timeout=5m
          image_id="$(kubectl get pods -l app=orders -o jsonpath='{.items[0].status.containerStatuses[0].imageID}')"
          test "${image_id#*@}" = "$DIGEST"
```

For this run, `steps.build.outputs.digest` resolves to `sha256:4ae0...9c1d`. The deployment sets that exact reference, waits for convergence, and refuses success unless a running pod's platform-reported `imageID` ends with the same digest.

**Success signal:** `kubectl rollout status` reports success and the final comparison matches `sha256:4ae0...9c1d`. The bounded external inputs are a pre-authenticated Kubernetes context for the `staging` environment and a deployment named `orders`; the environment administrator owns that credential and target mapping. `Unauthorized`, a missing context, or a different observed `imageID` fails the job.

**Not handled yet:** [deterministic build inputs](#3-unpinned-build-inputs-make-two-identical-builds-different), [artifact vs. cache lifetime](#4-a-cache-hit-is-not-a-production-artifact), [promotion mechanics and GitOps](#5-promotion-moves-metadata-never-rebuilds-bytes), and [verifying attestations before trusting a promoted digest](#7-what-promotion-gets-wrong-in-production).

---

[Production Pull-Request CI](02_pull_request_ci.md) covers getting a commit onto `main` in the first place. Optional background — not required to run the baseline above.

## 1. A Digest Is the Only Identity That Does Not Move

A deploy that reads `orders:latest` can pull a different image than the one your team just reviewed — the tag moved the moment someone else pushed. A rebuild that runs `docker build` again for "the same" release can also produce different bytes than what passed CI, even from identical source, if a base image or dependency resolved differently in between. Both failures share one cause: the identifier the deploy used was never tied to specific content.

```text
commit SHA       identifies source
image tag        helps humans find a build
image digest     identifies the exact container content
attestation      links content to build identity and process
SBOM             describes included components
```

An image tag such as `git-8f31c2a` is useful but may be overwritten unless the registry enforces immutability. A digest such as `sha256:4ae0...9c1d` is content-addressed: it is a hash of the actual bytes, so it changes if and only if the content does.

An **attestation** is a signed claim binding a digest to the workflow run that produced it — evidence of build identity and process, not of code quality. Its **SBOM** (software bill of materials) is the itemized list of packages and versions the image contains. **Provenance** is the build-origin evidence those two records combine to give a verifier: which source, workflow, and runner actually produced this digest. Promotion in this note always means moving a digest plus these records forward, never re-running the build.

> **Rule**: Promotion inputs should contain a digest or an equivalently immutable artifact identifier.

---

## 2. Build the Image Once, Promote the Same Digest Everywhere

> **Core:** the entire baseline is build once, capture the digest as a job output, then pass `image@digest` — never a tag — into every later job. Everything else in this section is what makes that safe to publish.

The following shape publishes an image to GitHub Container Registry and exposes its digest to later jobs.

> **Production:** the Docker and attestation actions use readable major tags below so the flow stays understandable while you learn it. Resolve each tag to a reviewed full commit SHA before adopting the workflow; full-SHA pinning is covered in [Workflow and Runner Hardening](../03_security_and_supply_chain/02_workflow_and_runner_hardening.md).

```yaml
name: Build Release Candidate

on:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: build-main-${{ github.ref }}
  cancel-in-progress: false

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
      artifact-metadata: write
    outputs:
      image: ${{ steps.image.outputs.name }}
      digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ github.token }}

      - name: Set up BuildKit
        uses: docker/setup-buildx-action@v3

      - name: Normalize image name
        id: image
        env:
          RAW_IMAGE: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        run: |
          set -euo pipefail
          image="$(printf '%s' "$RAW_IMAGE" | tr '[:upper:]' '[:lower:]')"
          echo "name=$image" >> "$GITHUB_OUTPUT"

      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          target: production
          push: true
          tags: |
            ${{ steps.image.outputs.name }}:git-${{ github.sha }}
          labels: |
            org.opencontainers.image.revision=${{ github.sha }}
            org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Attest build provenance
        uses: actions/attest@v4
        with:
          subject-name: ${{ steps.image.outputs.name }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true
          # push-to-registry additionally creates a storage record by default;
          # that write needs artifact-metadata: write above, or pass
          # create-storage-record: false to opt out instead.

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    permissions:
      contents: read
      id-token: write
    env:
      KUBE_CONTEXT: ${{ vars.KUBE_CONTEXT }}
    steps:
      - name: Authenticate to the reviewed staging cluster
        run: ./scripts/configure-kube-context.sh "$KUBE_CONTEXT"

      - name: Deploy image@digest to staging
        env:
          IMAGE: ${{ needs.build.outputs.image }}
          DIGEST: ${{ needs.build.outputs.digest }}
        run: |
          set -euo pipefail
          kubectl set image deployment/orders orders="${IMAGE}@${DIGEST}"
          kubectl rollout status deployment/orders --timeout=5m
          image_id="$(kubectl get pods -l app=orders -o jsonpath='{.items[0].status.containerStatuses[0].imageID}')"
          test "${image_id#*@}" = "$DIGEST"
```

`push-to-registry: true` above creates a storage record in the registry by default, which needs the `artifact-metadata: write` permission granted in `build`'s `permissions:` block — without it, the step fails once that default applies. If you don't want that record, pass `create-storage-record: false` instead of adding the permission. See [actions/attest usage](https://github.com/actions/attest#usage) and [Using artifact attestations](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations) (checked 2026-08-14).

The deploy job receives `image@digest`, never a mutable environment tag, and the Kubernetes control plane confirms the running digest. The environment administrator owns `KUBE_CONTEXT`, its identity, and cluster authorization; repository YAML alone cannot prove those settings. Extracting this into a shared, versioned reusable workflow — with its own permission ceiling and input contract — is the subject of [Reusable Workflows and Actions](04_reusable_workflows_and_actions.md).

For this run, `steps.build.outputs.digest` resolves to `sha256:4ae0...9c1d`, and the `deploy-staging` job logs `Deploying ghcr.io/acme/orders@sha256:4ae0...9c1d to staging`. Verifying the attestation before trusting that digest further confirms this workflow — not some other build — actually produced it:

```text
$ gh attestation verify oci://ghcr.io/acme/orders@sha256:4ae0...9c1d --owner acme
Loaded digest sha256:4ae0...9c1d for file://oci://ghcr.io/acme/orders@sha256:4ae0...9c1d
Loaded 1 attestation from GitHub API
✓ Verification succeeded!
```

---

## 3. Unpinned Build Inputs Make Two Identical Builds Different

Capture or constrain every input that can change output:

- dependency lockfiles and checksum verification;
- compiler and runtime versions;
- container base-image digests;
- build arguments;
- repository submodule commits;
- downloaded tools and actions;
- locale, timezone, and reproducibility flags where relevant.

```dockerfile
# A digest keeps the base stable even if the 3.12-slim tag moves.
FROM python:3.12-slim@sha256:REPLACE_WITH_VERIFIED_DIGEST AS production

WORKDIR /app
COPY requirements.txt .
RUN python -m pip install \
    --no-cache-dir \
    --require-hashes \
    --requirement requirements.txt

COPY src/ ./src/
USER 10001:10001
CMD ["python", "-m", "src.api"]
```

The digest in this teaching example must be replaced with one verified for the selected base. Automate digest update PRs so stable inputs do not become stale inputs — see [Dependency Update Governance](../03_security_and_supply_chain/04_dependency_update_governance.md) for how those PRs get reviewed, gated, and escalated once they open.

Bit-for-bit reproducibility is valuable but not always immediately achievable. At minimum, record inputs well enough to explain and rebuild the release under controlled conditions.

---

## 4. A Cache Hit Is Not a Production Artifact

| Storage | Use | Production identity? |
|---------|-----|----------------------|
| Actions cache | Dependency and build-layer acceleration | No |
| Workflow artifact | Reports or short-lived files between jobs | Usually no |
| Package registry | Versioned libraries and binaries | Yes, with digest/checksum |
| Container registry | Deployable images | Yes, by digest |
| Object storage release bucket | Signed archives and manifests | Yes, with checksum and immutability controls |

A cache hit must never change whether a build is correct. A deleted workflow run also deletes its attached workflow artifacts, so a production release should not depend solely on an Actions artifact with incidental retention.

---

## 5. Promotion Moves Metadata, Never Rebuilds Bytes

Promotion updates desired state:

```yaml
release:
  source:
    repository: acme/orders
    commit: 8f31c2a7d9...
    run_id: 8912345678
  artifact:
    image: ghcr.io/acme/orders
    digest: sha256:4ae0...9c1d
  target:
    environment: production
```

Common implementations:

| Model | Promotion action |
|-------|------------------|
| Direct deployment | Deployment workflow updates the platform to `image@digest` |
| **GitOps** — changing version-controlled desired state and letting a reconciler apply that reviewed change | A PR changes an environment manifest to the new digest |
| Release manifest | A signed manifest maps version and environment to digests |
| Registry promotion | Copy or retag by digest while preserving content identity |

> **Key insight**: promotion changes trusted metadata that points at immutable bytes; it does not rebuild those bytes. Every row in the table above is really just a different way to update that pointer safely.

> **Edge case:** if a registry copy changes the manifest digest — for example, due to media-type conversion — verify the resulting content and create new provenance for the transformed artifact.

---

## 6. A Deploy Log Is Not Enough for Incident Response

At minimum, retain:

- source repository, commit, and workflow run;
- artifact name, digest, SBOM, and attestation;
- build workflow version and runner type;
- deployment environment, timestamp, and actor;
- cloud deployment identifier;
- migration version;
- rollout result and verification links;
- previous known-good artifact.

This metadata supports incident response, vulnerability impact analysis, and recovery.

---

## 7. What Promotion Gets Wrong in Production

**Production rebuilds the source tag**

Staging and production no longer share the same candidate. Pass the digest from the build record.

**Only the human-readable tag is retained**

The tag later moves and the rollback target becomes ambiguous. Record the digest in deployment metadata.

**A successful push is treated as a successful build**

Scan, attest, and verify the produced content. Publication is a storage event, not quality evidence.

**Provenance is generated but never verified**

Attestation creation alone does not protect a consumer. Verify identity, repository, and signer policy before sensitive promotion or consumption.

---

## 8. References

- [Workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts)
- [Using artifact attestations](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations)
- [Publishing Docker images](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)
- [actions/attest usage](https://github.com/actions/attest#usage)

---

**Next**: [Reusable Workflows and Actions](04_reusable_workflows_and_actions.md)
