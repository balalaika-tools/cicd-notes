# SBOM, Provenance, and Artifact Attestations

> **Who this is for**: Teams that publish or deploy artifacts and need to answer what they contain, where they came from, and whether they were altered. Read [Workflow and Runner Hardening](02_workflow_and_runner_hardening.md) first.

---

## 1. Distinguish the Evidence

| Evidence | Answers |
|----------|---------|
| Artifact digest | Are these bytes exactly the referenced content? |
| SBOM | Which packages and components are present? |
| Vulnerability scan | Which known findings apply under this scanner's data and policy? |
| Provenance attestation | Which repository, commit, workflow, and build identity produced the artifact? |
| Signature | Did an accepted identity authorize this content or statement? |
| Release manifest | Which artifacts together form this product release? |

```text
source + dependencies + builder
              │
              ▼
       immutable artifact ─────> digest
              │
              ├───────────────> SBOM
              ├───────────────> provenance
              └───────────────> scan findings
                                      │
                                      ▼
                               policy decision
```

An SBOM is not proof of origin. A signature is not proof that content is safe. A scan is not a complete inventory. Use each for the decision it supports.

---

## 2. Generate Evidence in the Trusted Build

The workflow that publishes the final artifact should generate or bind:

- the artifact digest;
- the SBOM for the final artifact;
- provenance tied to the build workflow;
- source and build metadata;
- vulnerability-scan results.

Scanning only the source tree misses operating-system packages and build output. Generate the container SBOM from the published image or the exact local image content that produced its digest.

```yaml
permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write

steps:
  - name: Build and push image
    id: build
    uses: docker/build-push-action@v6 # Pin in production.
    with:
      push: true
      tags: ghcr.io/acme/orders:git-${{ github.sha }}

  - name: Generate an SPDX SBOM
    uses: anchore/sbom-action@v0 # Pin in production.
    with:
      image: ghcr.io/acme/orders@${{ steps.build.outputs.digest }}
      format: spdx-json
      output-file: sbom.spdx.json

  - name: Attest build provenance
    uses: actions/attest@v4 # Pin in production.
    with:
      subject-name: ghcr.io/acme/orders
      subject-digest: ${{ steps.build.outputs.digest }}
      push-to-registry: true

  - name: Attest the SBOM
    uses: actions/attest@v4 # Pin in production.
    with:
      subject-name: ghcr.io/acme/orders
      subject-digest: ${{ steps.build.outputs.digest }}
      sbom-path: sbom.spdx.json
      push-to-registry: true
```

The action versions shown are readable major tags. Resolve all of them to reviewed full commit SHAs in a production workflow.

---

## 3. Make the SBOM Useful

Prefer SPDX or CycloneDX, and retain:

- package name, version, and package URL where available;
- checksums;
- direct and transitive relationships;
- licenses;
- supplier or namespace;
- file information when the use case requires it;
- the artifact digest the SBOM describes.

An SBOM supports:

```text
new vulnerability announced
    ↓ query release SBOMs
    ↓ identify affected deployed digests
    ↓ prioritize by reachability and exposure
    ↓ rebuild, patch, or mitigate
```

Do not treat the SBOM as a one-time compliance attachment. Make it searchable and connect it to deployed inventory.

---

## 4. Understand GitHub Artifact Attestations

GitHub artifact attestations use workload identity to create signed claims about binaries or container images. Provenance includes information such as repository, commit SHA, workflow, triggering event, and environment.

For a container, the subject must be the image name without a tag plus the digest:

```yaml
- uses: actions/attest@v4 # Pin in production.
  with:
    subject-name: ghcr.io/acme/orders
    subject-digest: sha256:4ae0...9c1d
    push-to-registry: true
```

Availability varies by repository visibility and plan. At the time of writing, private/internal repository attestations require GitHub Enterprise Cloud, while supported public repositories can use them on current plans.

GitHub uses Sigstore for attestation signing. Public and private repositories use different Sigstore trust/storage arrangements; consumers should verify through supported tooling rather than assuming all bundles share the public transparency log.

---

## 5. Verify before Trusting

Binary:

```bash
gh attestation verify ./dist/orders-server \
  --repo acme/orders
```

Container:

```bash
gh attestation verify \
  oci://ghcr.io/acme/orders@sha256:4ae0...9c1d \
  --repo acme/orders
```

SPDX SBOM attestation:

```bash
gh attestation verify ./dist/orders-server \
  --repo acme/orders \
  --predicate-type https://spdx.dev/Document/v2.3
```

Verification policy should check:

- artifact digest;
- expected source repository and organization;
- expected workflow identity or trusted reusable builder;
- expected ref, environment, or event where supported;
- attestation predicate type;
- signer and transparency/trust root;
- revocation or incident state.

> **Principle**: Generating an attestation adds value only when a consumer verifies it against a policy.

---

## 6. Sign Release Manifests for Multi-Artifact Systems

A release often contains multiple images, charts, and migrations:

```yaml
release: 2.7.0
source_commit: 8f31c2a7d9...
artifacts:
  api:
    image: ghcr.io/acme/orders-api
    digest: sha256:4ae0...9c1d
  worker:
    image: ghcr.io/acme/orders-worker
    digest: sha256:82bd...7af0
  helm:
    chart: oci://ghcr.io/acme/charts/orders
    digest: sha256:71ee...021c
migrations:
  target: "2026_07_29_01"
```

Sign or attest the manifest, then deploy only artifact identities listed in it. This prevents a deployment from combining independently valid but incompatible components.

---

## 7. Scan at More than One Time

```text
pull request       → dependency change and source policy
build              → final artifact and image scan
registry           → rescan as vulnerability data changes
deployment         → block prohibited severity or provenance
runtime inventory  → find deployed affected digests
```

A clean build-time scan can become vulnerable tomorrow without a source change. Registry rescanning and deployed-digest inventory close that gap.

Define exception records with:

- finding identifier;
- affected digest or package;
- justification and compensating control;
- owner;
- approval;
- expiry.

---

## 8. Protect the Builder

Supply-chain evidence is only as credible as the builder boundary:

- use reviewed, pinned actions;
- build from the accepted commit;
- prevent untrusted PRs from using release credentials;
- isolate signing or attestation jobs;
- prefer ephemeral runners;
- protect workflow changes with code owners;
- do not let a called workflow elevate caller permissions;
- retain builder and audit logs.

For higher assurance, central reusable builders plus artifact attestations can support stronger SLSA build levels.

---

## 9. Common Failure Modes

**The SBOM describes the source environment**

It omits packages installed in the final image. Generate it from the release artifact.

**A tag is signed but the digest is not recorded**

The tag can later reference different content. Bind evidence to the digest.

**The attestation is stored beside an overwritable artifact**

Protect registry immutability and retention. Valid provenance for deleted content does not preserve availability.

**Only build-time scanning exists**

New vulnerability data never reaches deployed inventory. Rescan and map results to active digests.

---

## 10. References

- [Artifact attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations)
- [Using artifact attestations](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations)
- [Using attestations and reusable workflows for SLSA Build Level 3](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/increase-security-rating)
- [SLSA specification](https://slsa.dev/spec/)
- [CycloneDX specification](https://cyclonedx.org/specification/overview/)
- [SPDX specification](https://spdx.dev/use/specifications/)

---

**Next**: [Environments and Artifact Promotion](../04_delivery_operations/01_environments_and_promotions.md)
