# Permissions, Secrets, and OIDC

> **Who this is for**: Engineers securing workflow identity and cloud deployment access. Read [GitHub Actions Performance and Reliability](../02_github_actions/06_performance_and_reliability.md) first.

---

## 1. Separate the Identities

A workflow may use several identities with different trust boundaries:

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
| Fine-grained PAT | User-bound cross-repository fallback | Configured repositories, permissions, and expiry |
| OIDC token | Exchange workflow claims for cloud credentials | Short-lived signed identity assertion |
| Environment secret | Legacy credential or non-federated secret | Stored until rotated; released after environment gates |

Prefer the narrowest identity that represents the automation itself.

---

## 2. Make `GITHUB_TOKEN` Read-Only by Default

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
      - run: ./scripts/test.sh

  publish:
    if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    permissions:
      contents: read
      packages: write
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/publish.sh
```

Specifying any permission changes unspecified permissions to `none`. Declare the workflow baseline, then elevate only the job that needs a write.

Avoid:

```yaml
permissions: write-all
```

An action can access `github.token` even when the token is not passed as an explicit input. Token safety depends on the job's `permissions`, action trust, and event—not merely on whether `${{ secrets.GITHUB_TOKEN }}` appears in YAML.

---

## 3. Know the `GITHUB_TOKEN` Boundary

`GITHUB_TOKEN` is scoped to the repository where the workflow runs. Use it for:

- checking out private content in the same repository;
- publishing packages owned by that repository when permission is granted;
- updating checks, pull requests, or issues in that repository;
- requesting artifact attestations when declared permissions allow it.

Use a GitHub App for selected cross-repository access.

Events created with `GITHUB_TOKEN` generally do not recursively trigger new workflow runs. `workflow_dispatch` and `repository_dispatch` are explicit exceptions, but the repository scope still applies. Cross-repository dispatch requires a token that can access the destination.

---

## 4. Store Secrets at the Narrowest Level

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

Rules:

- never print, serialize, upload, or cache secrets;
- do not pass secrets through job outputs or command-line arguments when an environment variable or file descriptor is safer;
- transform secrets as little as possible—automatic redaction is not guaranteed for every encoding;
- rotate on exposure, staff change, or policy schedule;
- delete unused secrets and audit access;
- prefer a GitHub App over a personal access token;
- prefer OIDC over permanent cloud access keys.

Forked pull-request workflows do not receive normal repository secrets. Do not design ordinary PR validation to depend on them.

---

## 5. OIDC Replaces Stored Cloud Keys

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

---

## 6. Use Environment-Bound AWS Trust

Workflow:

```yaml
jobs:
  deploy:
    environment: production
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    steps:
      # Resolve this readable major tag to a reviewed full SHA in production.
      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: arn:aws:iam::123456789012:role/orders-production-deploy
          aws-region: eu-west-1
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

⚠️ Repositories created after July 15, 2026 use immutable default OIDC subjects containing the owner and repository IDs. An example shape is:

```text
repo:acme@123456/orders@789012:environment:production
```

Older repositories keep the previous subject unless they opt in. Inspect an actual token or the repository OIDC settings before writing the cloud trust condition; do not blindly copy either example.

AWS does not support GitHub's custom OIDC claims, so AWS policies normally constrain standard `aud` and `sub`. Other providers may additionally enforce claims such as `job_workflow_ref`.

---

## 7. Separate Trust Policy from Permission Policy

The trust policy answers:

> Which GitHub workload may assume this role?

The permission policy answers:

> What may the assumed role do?

Example deployment permission concept:

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

## 8. Bind Trust to Reusable Workflows When Supported

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

Cloud providers that support the claim—or support a customized subject containing it—can require deployments to pass through the centrally governed workflow.

Permissions still originate from the caller and cannot be elevated through nesting.

---

## 9. Prefer GitHub Apps for GitHub-to-GitHub Automation

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

## 10. Common Failure Modes

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

---

**Next**: [Workflow and Runner Hardening](02_workflow_and_runner_hardening.md)
