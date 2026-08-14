# Permissions, Secrets, and OIDC

> **Who this is for**: Engineers securing workflow identity and cloud deployment access.

## The short version

A repository secret holding one permanent AWS key gives every job in every workflow run — including a pull-request test job anyone can trigger — the same reach as your deploy job. **OIDC** (OpenID Connect — a signed workload-identity token GitHub issues per job, which a cloud provider exchanges for short-lived credentials) replaces that stored key: nothing long-lived sits in a secrets store waiting to leak. The cloud validates the token's claims — which repository, ref, and environment issued it — against a trust policy you write, then hands back credentials that expire in about an hour. Which jobs can reach production is therefore a cloud-side policy decision, not a line of workflow YAML.

**What you need (4 things):**

1. `id-token: write` permission on the job that requests the token.
2. An OIDC identity provider registered in the cloud account (`token.actions.githubusercontent.com` for AWS).
3. A cloud role trust policy that checks the token's `aud` and `sub` claims.
4. An action that exchanges the token for credentials — `aws-actions/configure-aws-credentials`, shown below.

**The code:**

```yaml
permissions:
  id-token: write   # request the OIDC token; nothing else needed for this step
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@e6de054238d6b7531b4efff3b6587d9aade6a06c # v6.2.3
        with:
          role-to-assume: arn:aws:iam::123456789012:role/orders-deploy
          aws-region: eu-west-1
      - run: aws sts get-caller-identity
```

**Success signal:** `aws sts get-caller-identity` prints `"Account": "123456789012"` and an `Arn` ending in `assumed-role/orders-deploy/<session>` — the account and role this job should have reached. A token whose claims don't satisfy the trust policy never gets this far: `configure-aws-credentials` fails first.

**Not handled yet:** [environment protection](#6-bind-cloud-trust-to-the-exact-repository-environment-and-ref), [claim binding](#8-reusable-workflow-claims-let-cloud-trust-follow-the-workflow-not-just-the-caller), [role and permission scoping](#7-a-narrow-trust-policy-with-a-broad-permission-policy-is-still-broad), [secret rotation](#4-secret-scope-should-match-blast-radius-not-convenience), and [cross-repository automation](#9-cross-repository-automation-needs-its-own-identity-not-a-personal-one).

---

For workflow trigger and event-context background, see [GitHub Actions Performance and Reliability](../02_github_actions/06_performance_and_reliability.md) — not required for the baseline above.

---

## 1. One Broad Credential Reaches Everywhere It's Given To

Say `orders`'s only AWS credential is a repository secret, `AWS_SECRET_ACCESS_KEY` — a permanent IAM user key with full deploy access to production. Every job in every workflow run can read it via `${{ secrets.AWS_SECRET_ACCESS_KEY }}`: the pull-request `test` job from a first-time contributor as easily as the `main`-branch `deploy` job. A test job that shells out to `aws` for a fixture, or a workflow with a permissive `if:` condition, hands that same production key to code nobody has reviewed. GitHub has no concept of "this secret is only for the deploy job" — access follows what the workflow file grants, not what a human intended.

The fix is to stop treating "the workflow's identity" as one thing. A workflow run actually carries several distinct identities with different trust boundaries, and picking the narrowest one per job is what keeps a compromised test job from reaching production:

```text
GitHub event
    │
    ▼
GITHUB_TOKEN
    repository-scoped GitHub API identity
    │
    ├── GitHub App token
    │     selected cross-repository GitHub access
    │
    └── OIDC token
          signed workload identity claims
              │
              ▼
        short-lived cloud role
```

| Identity | Use | Lifetime and scope |
|----------|-----|--------------------|
| `GITHUB_TOKEN` | Current-repository GitHub API and packages | Created per job; repository-scoped |
| GitHub App installation token | Selected repositories or organization resources | Short-lived; app installation and permissions |
| Fine-grained **PAT** (personal access token — bound to one user account, revoked the moment they rotate or lose it) | User-bound cross-repository fallback | Configured repositories, permissions, and expiry |
| OIDC token | Exchange workflow claims for cloud credentials | Short-lived signed identity assertion |
| Environment secret | Legacy credential or non-federated secret | Stored until rotated; released after environment gates |

Prefer the narrowest identity that represents the automation itself.

> **Core:** picking the narrowest identity for each job is the one decision that matters before anything else in this note. Everything from here is how to implement that narrowing for a specific identity type.

---

## 2. Declare `contents: read`, Then Elevate Only the Job That Writes

An unscoped `GITHUB_TOKEN` can carry write access to contents, packages, issues, and more in the current repository — enough for a compromised dependency inside `test.sh` to push a tag or cut a release nobody asked for. Declaring `permissions` at the workflow level fixes that, and checking out the repository is a prerequisite both jobs below actually need:

```yaml
name: Secure workflow

on:
  pull_request:

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - run: ./scripts/test.sh

  publish:
    if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    permissions:
      contents: read
      packages: write
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - run: ./scripts/publish.sh
```

`persist-credentials: false` stops `actions/checkout` from writing `GITHUB_TOKEN` into the local git config after checkout. Left at its default (`true`), any later step — or code that `test.sh`/`publish.sh` pulls in — can read that token straight back out of `.git/config`.

Specifying any permission changes unspecified permissions to `none`. Declare the workflow baseline, then elevate only the job that needs a write.

Avoid:

```yaml
permissions: write-all
```

An action can access `github.token` even when the token is not passed as an explicit input. Token safety depends on the job's `permissions`, action trust, and event — not merely on whether `${{ secrets.GITHUB_TOKEN }}` appears in YAML.

---

## 3. `GITHUB_TOKEN` Never Leaves Its Own Repository

`GITHUB_TOKEN` is scoped to the repository where the workflow runs. Use it for:

- checking out private content in the same repository;
- publishing packages owned by that repository when permission is granted;
- updating checks, pull requests, or issues in that repository;
- requesting artifact attestations when declared permissions allow it.

Use a GitHub App for selected cross-repository access.

Events created with `GITHUB_TOKEN` generally do not recursively trigger new workflow runs — a push made with `GITHUB_TOKEN` inside a workflow won't itself start another workflow run, which stops accidental infinite loops. `workflow_dispatch` and `repository_dispatch` are explicit exceptions; so is a `pull_request` run opened, synchronized, or reopened by `GITHUB_TOKEN` — GitHub creates that run but holds it pending approval rather than skipping it outright ([Trigger a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow), checked 2026-08-14). All three still keep the repository scope: cross-repository dispatch requires a token that can access the destination.

---

## 4. Secret Scope Should Match Blast Radius, Not Convenience

```text
organization secret
    shared by selected repositories

repository secret
    available to eligible workflows in one repository

environment secret
    available only to jobs referencing that environment
    and only after its protection rules pass
```

Use environment secrets for environment-specific legacy values. Use organization secrets only when the same secret genuinely must be shared.

Immediate baseline — apply these to every secret before the workflow ships:

- never print, serialize, upload, or cache secrets;
- do not pass secrets through job outputs or command-line arguments when an environment variable or file descriptor is safer;
- transform secrets as little as possible — automatic redaction is not guaranteed for every encoding;
- prefer a GitHub App over a personal access token;
- prefer OIDC over permanent cloud access keys.

The command-line-argument rule in practice:

```yaml
      - name: Call internal API
        env:
          API_TOKEN: ${{ secrets.INTERNAL_API_TOKEN }}
        run: curl -sf -H "Authorization: Bearer $API_TOKEN" https://internal.example.com/health
```

The token reaches `curl` through an environment variable, not a `-H` string built inline in `run:` from `${{ secrets.INTERNAL_API_TOKEN }}`. GitHub's log redaction masks the exact secret value it already knows about — a value re-encoded or concatenated into a longer string can slip past that match.

> **Production:** the remaining two rules are an ongoing program, not a one-time setup — rotate on exposure, staff change, or policy schedule; delete unused secrets and audit access.

Forked pull-request workflows do not receive normal repository secrets. Do not design ordinary PR validation to depend on them.

---

## 5. OIDC Trades a Stored Key for a Token Nobody Can Steal at Rest

```text
GitHub Actions job
    │ requests OIDC token (`id-token: write`)
    ▼
GitHub OIDC issuer
    │ signs claims: repository, ref/environment, workflow, run, runner...
    ▼
Cloud security token service
    │ validates issuer + audience + subject conditions
    ▼
Short-lived role credentials
    │
    ▼
Only the deployment API operations allowed by the role policy
```

`id-token: write` permits the job to request an OIDC token. It does not directly grant cloud access. The cloud trust policy decides which claims may assume which role, and the role permission policy decides what that role may do.

> **Edge case:** not every target can validate a GitHub-issued token — an on-prem deployment endpoint or a SaaS API that only accepts static API keys has no OIDC-aware equivalent of AWS's STS to exchange it with. There, fall back to a secret scoped to one repository or environment, used by exactly one workflow, and rotated on a fixed schedule — never a broad, account-wide key. That fallback still carries a bigger exposure window than OIDC: the credential sits at rest between rotations, so a leaked key stays valid until someone notices and rotates it, where a leaked OIDC-derived credential expires on its own within roughly an hour.

---

## 6. Bind Cloud Trust to the Exact Repository, Environment, and Ref

Say the AWS trust policy for `orders-production-deploy` accepts any token where `aud` is `sts.amazonaws.com` and `sub` starts with `repo:acme/` — written broadly so any team's workflow can assume it without a change request. An attacker who can trigger a run in *any* `acme` repository — a low-traffic internal tool, a feature branch with no protection, a fork workflow still running on an unprotected ref — requests an OIDC token from that run instead. Its `sub` claim (`repo:acme/some-other-repo:ref:refs/heads/feature-x`) still starts with `repo:acme/`, so the condition matches. AWS's STS validates the signature, sees the match, and returns `orders-production-deploy`'s credentials — full production access, reached without the attacker ever touching a secret. The trust policy did exactly what it was configured to do; it was configured to trust too much.

Binding the condition to the repository *and* an environment closes that gap:

```yaml
jobs:
  deploy:
    environment: production
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@e6de054238d6b7531b4efff3b6587d9aade6a06c # v6.2.3
        with:
          role-to-assume: arn:aws:iam::123456789012:role/orders-production-deploy
          aws-region: eu-west-1
      - name: Verify assumed identity
        run: aws sts get-caller-identity
```

`configure-aws-credentials@v6` requires the Node 24 runtime; `ubuntu-latest` and current self-hosted runner images already provide it, but an older pinned runner image needs updating first ([release notes](https://github.com/aws-actions/configure-aws-credentials/releases), checked 2026-08-14).

> **Core:** always run the verification step. This job should land in account `123456789012` as `orders-production-deploy` — confirm that before trusting the policy below to be as restrictive as it looks:

```json
{
    "UserId": "AROAEXAMPLE:GitHubActions",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/orders-production-deploy/GitHubActions"
}
```

AWS role trust policy for a repository using the historical name-based default OIDC subject:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:acme/orders:environment:production"
        }
      }
    }
  ]
}
```

When the subject is environment-based, the branch is not included in the default `sub`. Protect the GitHub environment so only the intended branches or tags may deploy.

Confirm the condition actually rejects what it should, rather than assuming it does: trigger the same job from a context it's meant to reject — a different repository, or an unprotected branch if the role is bound to `environment:production` — and expect `configure-aws-credentials` to fail before `get-caller-identity` ever runs:

```text
Error: Credentials could not be loaded, please check your action inputs: An error occurred (AccessDenied) when calling the AssumeRoleWithWebIdentity operation: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

⚠️ If that same attempt succeeds instead, the condition is broader than you intended — that's the silent failure. Nothing in the logs looks wrong; the job just quietly has more reach than the environment gate was supposed to give it.

> **Edge case:** repositories created after July 15, 2026 use immutable default OIDC subjects containing the owner and repository IDs instead of names, for example:
>
> ```text
> repo:acme@123456/orders@789012:environment:production
> ```
>
> Older repositories keep the previous subject unless they opt in. Inspect an actual token or the repository's OIDC settings before writing the cloud trust condition; don't copy either example blindly.

AWS does not support GitHub's custom OIDC claims, so AWS policies normally constrain standard `aud` and `sub`. Other providers may additionally enforce claims such as `job_workflow_ref`.

---

## 7. A Narrow Trust Policy With a Broad Permission Policy Is Still Broad

The trust policy answers:

> Which GitHub workload may assume this role?

The permission policy answers:

> What may the assumed role do?

> **Key insight**: a cloud role's security depends on two independent axes, and narrowing only one leaves the compromise blast radius where it was. A narrowly-trusted role with an admin permission policy is still an admin-level compromise; a role scoped to one ECS service but trusted by every repository still hands that service to whichever workflow gets compromised first.

The policy below lets the deploy role update exactly one **ECS** (Elastic Container Service — AWS's managed container-orchestration service) service and pass exactly two task roles to it:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UpdateOnlyOrdersService",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:UpdateService"
      ],
      "Resource": "arn:aws:ecs:eu-west-1:123456789012:service/production/orders"
    },
    {
      "Sid": "PassOnlyOrdersTaskRoles",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::123456789012:role/orders-task",
        "arn:aws:iam::123456789012:role/orders-execution"
      ],
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "ecs-tasks.amazonaws.com"
        }
      }
    }
  ]
}
```

In a real ECS design, additional actions may be required to register and inspect task definitions. Add only the resource-scoped actions used by the deployment implementation.

Use different roles for staging, production, infrastructure, migrations, and artifact publication. A compromised test job should not inherit production authority.

---

## 8. Reusable-Workflow Claims Let Cloud Trust Follow the Workflow, Not Just the Caller

> **Edge case:** only relevant if your cloud provider validates `job_workflow_ref` or a subject built from it. AWS, per the previous section, does not — check your provider before designing around this.

OIDC tokens for jobs inside reusable workflows include `job_workflow_ref` and `job_workflow_sha`.

```text
caller claims:
    repository = acme/orders
    ref         = refs/heads/main
    environment = production

called workflow claim:
    job_workflow_ref =
      acme/platform-workflows/.github/workflows/deploy.yml@refs/tags/v3
```

Cloud providers that support the claim — or support a customized subject containing it — can require deployments to pass through the centrally governed workflow.

Permissions still originate from the caller and cannot be elevated through nesting.

---

## 9. Cross-Repository Automation Needs Its Own Identity, Not a Personal One

A GitHub App:

- acts as automation rather than as an employee;
- can be installed only on selected repositories;
- has declared, fine-grained permissions;
- creates short-lived installation tokens;
- produces a distinct audit identity.

```yaml
- id: app-token
  uses: actions/create-github-app-token@v3 # Pin in production.
  with:
    client-id: ${{ vars.CICD_APP_CLIENT_ID }}
    private-key: ${{ secrets.CICD_APP_PRIVATE_KEY }}
    owner: acme
    repositories: platform-deployments
```

Restrict where the private key is available. If every repository can mint the app token, the app's narrow installation is no longer a meaningful repository boundary.

---

## 10. What Still Breaks After Every Rule Above Is Followed

**OIDC trusts an entire organization**

Any repository may assume production. Restrict repository identity, environment or ref, audience, and where possible the approved reusable workflow.

**The role trust is narrow but its permission policy is admin**

Compromise still has full impact. Scope both assumption and actions.

**Environment approval exists, but a different workflow can target it**

Protect workflow changes with code ownership and bind cloud trust to the strongest supported workflow claims.

**Secrets are masked, so logging is assumed safe**

Derived, encoded, or structured secret values may not redact. Prevent output rather than relying on redaction.

---

## 11. References

- [`GITHUB_TOKEN`](https://docs.github.com/en/actions/concepts/security/github_token)
- [Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
- [OpenID Connect reference](https://docs.github.com/en/actions/reference/security/oidc)
- [Configuring OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [GitHub App authentication in Actions](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/making-authenticated-api-requests-with-a-github-app-in-a-github-actions-workflow)
- [Trigger a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow)
- [`configure-aws-credentials` releases](https://github.com/aws-actions/configure-aws-credentials/releases)

---

**Next**: [Workflow and Runner Hardening](02_workflow_and_runner_hardening.md)
