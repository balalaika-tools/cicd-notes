# SBOM, Provenance, and Artifact Attestations

> **Who this is for**: Teams that publish or deploy artifacts and need to answer what they contain, where they came from, and whether they were altered.

## The short version

A container image tag is a movable pointer — `ghcr.io/acme/orders:latest` can point at different bytes an hour from now, so trusting the tag alone tells a deploy pipeline nothing about what it's actually about to run. An **artifact digest** (the immutable `sha256:...` hash of the exact bytes) fixes *what* you're trusting, and a **provenance attestation** — a signed, machine-readable claim, cryptographically bound to that digest, stating which repository, commit, and workflow built it — fixes *where it came from*. Binding both to one digest and having a consumer check them before deploy turns "we build in CI" from a claim in a runbook into something a machine enforces on every release.

**What you need (3 things):**

1. An artifact published with its digest captured at build time, not re-derived later.
2. A provenance attestation generated inside that same trusted build job, bound to that exact digest.
3. A verification step that checks the attestation's repository, workflow, and digest before anything treats the artifact as trusted.

**The code:**

```yaml
permissions:
  id-token: write
  attestations: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - id: build
        uses: docker/build-push-action@53b7df96c91f9c12dcc8a07bcb9ccacbed38856a # v7.3.0
        with:
          push: true
          tags: ghcr.io/acme/orders:git-${{ github.sha }}
      - uses: actions/attest@1e69f48acb82d1966a394da916b4c1698aa569d6 # v4.2.2
        with:
          subject-name: ghcr.io/acme/orders
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true
```

**Success signal:** verifying the digest that job just pushed returns a policy match and exits `0` — see [Verify Before Trusting the Artifact](#5-verify-before-trusting-the-artifact) for the exact output on match and on a swapped or wrong-repository artifact.

**Not handled yet:** [SBOM depth](#3-make-the-sbom-useful), [multi-artifact manifests](#6-sign-release-manifests-for-multi-artifact-systems), [rescanning](#7-scan-at-more-than-one-time), [builder protection](#8-protect-the-builder), and [verification policy hardening](#5-verify-before-trusting-the-artifact).

---

For the token and runner boundaries the build job above depends on, see [Workflow and Runner Hardening](02_workflow_and_runner_hardening.md) — read after this baseline, not before it.

> **Core:** binding evidence to the exact digest — never the tag — and having a consumer actually verify it before deploy is the non-negotiable baseline. Section 2 shows the full runnable job with registry auth; sections 3–8 harden and extend it.

---

## 1. Distinguish the Evidence

Say a deploy pipeline is about to promote `ghcr.io/acme/orders@sha256:4ae0...9c1d` to production. It checks three things before it does: does a **software bill of materials**, or **SBOM** — a machine-readable inventory of every package and component in the artifact — exist for exactly this digest? Does a provenance attestation also name this digest, and does its repository and workflow match what the deploy policy expects? All three pass, and the pipeline promotes the image. Any one is missing, wrong, or names a different digest, and the pipeline blocks the release before it ever reaches a cluster that can serve traffic.

That single decision generalizes to six kinds of evidence, each answering a different question a release process needs answered before it trusts an artifact:

| Evidence | Answers | Used above |
|----------|---------|:---:|
| **Artifact digest** | Are these bytes exactly the referenced content? | ★ |
| **SBOM** | Which packages and components are present? | ★ |
| Vulnerability scan | Which known findings apply under this scanner's data and policy? | |
| **Provenance attestation** | Which repository, commit, workflow, and build identity produced the artifact? | ★ |
| Signature | Did an accepted identity authorize this content or statement? | |
| Release manifest | Which artifacts together form this product release? | |

★ marks the three evidence types the trace above actually used to decide; the rest answer questions the later sections cover — vulnerability policy, signing, and multi-artifact releases.

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
name: Build, attest, and publish

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      - name: Log in to GHCR
        uses: docker/login-action@dbcb813823bdd20940b903addbd779551569679f # v4.6.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }} # scoped to this repo and run, not a stored PAT

      - name: Build and push image
        id: build
        uses: docker/build-push-action@53b7df96c91f9c12dcc8a07bcb9ccacbed38856a # v7.3.0
        with:
          push: true
          tags: ghcr.io/acme/orders:git-${{ github.sha }}

      - name: Generate an SPDX SBOM
        uses: anchore/sbom-action@e22c389904149dbc22b58101806040fa8d37a610 # v0.24.0
        with:
          image: ghcr.io/acme/orders@${{ steps.build.outputs.digest }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Attest build provenance
        uses: actions/attest@1e69f48acb82d1966a394da916b4c1698aa569d6 # v4.2.2
        with:
          subject-name: ghcr.io/acme/orders
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true

      - name: Attest the SBOM
        uses: actions/attest@1e69f48acb82d1966a394da916b4c1698aa569d6 # v4.2.2
        with:
          subject-name: ghcr.io/acme/orders
          subject-digest: ${{ steps.build.outputs.digest }}
          sbom-path: sbom.spdx.json
          push-to-registry: true

      - name: Verify what was just published
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh attestation verify \
            oci://ghcr.io/acme/orders@${{ steps.build.outputs.digest }} \
            --repo ${{ github.repository }}
```

Every action above is pinned to the reviewed full commit SHA behind its release tag — the trailing comment is for humans and Dependabot/Renovate, not the resolver. See [Only a Full Commit SHA Actually Pins an Action](02_workflow_and_runner_hardening.md#2-only-a-full-commit-sha-actually-pins-an-action) for why a movable tag defeats this entire evidence chain even when every step above is otherwise correct: an attacker who moves one of these tags after review gets executed with the release job's `packages: write` and `attestations: write` authority, the same as any other step.

The final step matters as much as the four before it — the workflow above both generates the evidence and confirms, in the same run, that the evidence it just produced actually verifies. A pipeline that only generates never proves the attestation is retrievable or well-formed until the day a consumer needs it.

---

## 3. Make the SBOM Useful

Prefer **SPDX** (Software Package Data Exchange, an SBOM format hosted by the Linux Foundation) or **CycloneDX** (an OWASP-maintained SBOM format with stronger built-in support for vulnerability and license data), and retain:

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
- uses: actions/attest@1e69f48acb82d1966a394da916b4c1698aa569d6 # v4.2.2
  with:
    subject-name: ghcr.io/acme/orders
    subject-digest: sha256:4ae0...9c1d
    push-to-registry: true
```

Availability varies by repository visibility and plan. At the time of writing, private/internal repository attestations require GitHub Enterprise Cloud, while supported public repositories can use them on current plans.

GitHub uses Sigstore for attestation signing. Public and private repositories use different Sigstore trust/storage arrangements; consumers should verify through supported tooling rather than assuming all bundles share the public transparency log.

> **Edge case:** GitHub-hosted attestations assume consumers are willing to trust GitHub's own identity and API availability as part of the verification path. That doesn't hold everywhere — a consumer that needs forge-neutral trust (verification that doesn't depend on GitHub as a party), fully offline verification with no call to the GitHub API at deploy time, or a signing identity governed by an organization other than the one operating the repository should sign and verify with a standalone Sigstore/cosign workflow instead, where your own policy — not GitHub's API — decides what's trusted.

---

## 5. Verify Before Trusting the Artifact

Say the deploy pipeline instead pulled by the moving tag `ghcr.io/acme/orders:latest` rather than the digest. An attacker who gains push access to that repository — a compromised publish credential, a takeover of a build dependency, or a stolen token — retags `latest` to point at an image built outside the trusted workflow. The tag string never changes; `docker pull ghcr.io/acme/orders:latest` simply returns different bytes than it did an hour ago. A consumer that only checks "does *a* provenance attestation exist for `acme/orders`," without binding that check to the exact digest it is about to run, accepts the substitution — the attacker's image deploys with whatever authority the deployment carries. An attestation generated for the old digest never covers the new one, so binding verification to the digest is what turns "an attestation exists somewhere for this repository" into "an attestation exists for *this exact* `sha256:...`, built by *this* repository and workflow" — the only check that actually rejects the swap.

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

`--predicate-type` names the schema of the attested claim — the default is `https://slsa.dev/provenance/v1` (build provenance), so an SBOM attestation needs its own predicate type set explicitly or the command looks for the wrong kind of claim and reports none found.

Verification policy should check:

- artifact digest;
- expected source repository and organization;
- expected workflow identity or trusted reusable builder;
- expected ref, environment, or event where supported;
- attestation predicate type;
- signer and transparency/trust root;
- revocation or incident state.

> **Production:** the checks above are a policy, not a one-time command. Encode them in whatever gate runs before deploy — a required check, an admission controller, a release script — so a human forgetting a flag one time doesn't skip them.

**How you know it's working:** running the container command above against a digest that really was built and attested by `acme/orders` prints the enforced policy criteria, then the fields that matched, and exits `0`:

```text
$ gh attestation verify oci://ghcr.io/acme/orders@sha256:4ae0...9c1d --repo acme/orders
Loaded digest sha256:4ae0...9c1d for oci://ghcr.io/acme/orders@sha256:4ae0...9c1d
Loaded 1 attestation from GitHub API

The following policy criteria will be enforced:
- Predicate type must match:................ https://slsa.dev/provenance/v1
- Source Repository Owner URI must match:... https://github.com/acme
- Source Repository URI must match:......... https://github.com/acme/orders
- Subject Alternative Name must match regex: (?i)^https://github.com/acme/orders/
- OIDC Issuer must match:................... https://token.actions.githubusercontent.com

✓ Verification succeeded!

The following 1 attestation matched the policy criteria

- Attestation #1
  - Build repo:..... acme/orders
  - Build workflow:. .github/workflows/release.yml@refs/heads/main
```

Exit status `0`. Read the printed repository and workflow fields, not just the checkmark — a match on repository alone with an unexpected workflow means the policy is looser than it looks.

⚠️ A substituted artifact or the wrong `--repo` doesn't print a warning buried in otherwise-normal output — the command fails outright, with no policy table at all:

```text
$ gh attestation verify oci://ghcr.io/acme/orders@sha256:4ae0...9c1d --repo acme/other-repo
Loaded digest sha256:4ae0...9c1d for oci://ghcr.io/acme/orders@sha256:4ae0...9c1d
✗ Loading attestations from GitHub API failed

Error: HTTP 404: Not Found (https://api.github.com/repos/acme/other-repo/attestations/sha256:4ae0...9c1d...)
```

Exit status `1`. Treat any nonzero exit as "do not deploy" in a script — never grep the output for a specific error string, since GitHub can change the wording without notice.

> **Key insight**: an attestation is only as valuable as the policy a consumer actually enforces against it — generating one without a verification gate is, to everything downstream, indistinguishable from never having generated it at all. The evidence and the check that reads it are one mechanism, not two independent features.

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

For higher assurance, central reusable builders plus artifact attestations can support stronger **SLSA** (Supply-chain Levels for Software Artifacts — a specification that grades how tamper-resistant a build process is) build levels.

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
- [Publishing Docker images](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)
- [`gh attestation verify` manual](https://cli.github.com/manual/gh_attestation_verify)
- [SLSA specification](https://slsa.dev/spec/)
- [CycloneDX specification](https://cyclonedx.org/specification/overview/)
- [SPDX specification](https://spdx.dev/use/specifications/)
- [docker/build-push-action releases](https://github.com/docker/build-push-action)
- [docker/login-action](https://github.com/docker/login-action)
- [anchore/sbom-action](https://github.com/anchore/sbom-action)
- [actions/attest](https://github.com/actions/attest)

---

**Next**: [Dependency Update Governance](04_dependency_update_governance.md)
