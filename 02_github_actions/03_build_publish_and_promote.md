# Build, Publish, and Promote

> **Who this is for**: Teams turning an accepted commit into an immutable release candidate. Read [Production Pull-Request CI](02_pull_request_ci.md) first.

---

## 1. Use the Digest as Release Identity

```text
commit SHA       identifies source
image tag        helps humans find a build
image digest     identifies the exact container content
attestation      links content to build identity and process
SBOM             describes included components
```

An image tag such as `git-8f31c2a` is useful but may be overwritten unless the registry enforces immutability. A digest such as `sha256:4ae0...9c1d` is content-addressed.

> **Rule**: Promotion inputs should contain a digest or an equivalently immutable artifact identifier.

---

## 2. Build and Publish Once

The following shape publishes an image to GitHub Container Registry and exposes its digest to later jobs.

> The Docker and attestation actions use readable major tags below so the flow remains understandable. Resolve each tag to a reviewed full commit SHA before adopting the workflow; full-SHA pinning is covered in [Workflow and Runner Hardening](../03_security_and_supply_chain/02_workflow_and_runner_hardening.md).

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

  deploy-staging:
    needs: build
    uses: ./.github/workflows/reusable-deploy.yml
    permissions:
      contents: read
      id-token: write
    with:
      environment: staging
      image: ${{ needs.build.outputs.image }}
      digest: ${{ needs.build.outputs.digest }}
```

The deploy job receives `image@digest`, never a mutable environment tag.

---

## 3. Keep Build Inputs Deterministic

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

The digest in this teaching example must be replaced with one verified for the selected base. Automate digest update PRs so stable inputs do not become stale inputs.

Bit-for-bit reproducibility is valuable but not always immediately achievable. At minimum, record inputs well enough to explain and rebuild the release under controlled conditions.

---

## 4. Separate Artifacts from Caches

| Storage | Use | Production identity? |
|---------|-----|----------------------|
| Actions cache | Dependency and build-layer acceleration | No |
| Workflow artifact | Reports or short-lived files between jobs | Usually no |
| Package registry | Versioned libraries and binaries | Yes, with digest/checksum |
| Container registry | Deployable images | Yes, by digest |
| Object storage release bucket | Signed archives and manifests | Yes, with checksum and immutability controls |

A cache hit must never change whether a build is correct. A deleted workflow run also deletes its attached workflow artifacts, so a production release should not depend solely on an Actions artifact with incidental retention.

---

## 5. Promote Metadata, Not Bytes

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
| GitOps | A PR changes an environment manifest to the new digest |
| Release manifest | A signed manifest maps version and environment to digests |
| Registry promotion | Copy or retag by digest while preserving content identity |

If a registry copy changes the manifest digest—for example, due to media-type conversion—verify the resulting content and create new provenance for the transformed artifact.

---

## 6. Record Release Metadata

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

## 7. Promotion Failure Modes

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

---

**Next**: [Reusable Workflows and Actions](04_reusable_workflows_and_actions.md)
