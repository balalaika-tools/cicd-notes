# GitHub App Provisioning and Credential Lifecycle

> **Who this is for**: Administrators creating an automation identity that must act across selected repositories without borrowing a person's token.

## Start With One Operation Against One Repository

The `cicd-dispatch` App may create repository dispatches in `acme/platform-deployments` and read metadata there. It cannot access `acme/payments`. That boundary comes from registration permissions and installation scope, not from the workflow that later requests a token.

```yaml
app: cicd-dispatch
owner: platform-security
permissions:
  contents: write
  metadata: read
installation:
  selection: selected
  repositories: [acme/platform-deployments]
private_key_custodian: production-secret-manager
rotation_days: 90
```

**Success signal:** an installation token can dispatch to `platform-deployments`; the same request to `payments` returns `404` or authorization denied.

---

## 1. Choose Permissions From the API Call Backward

> **Core:** derive registered permissions from one required API operation, then restrict the installation to the repositories that need it.

Start from the exact endpoint and reduce permissions until it fails, then grant only the documented minimum. Do not add `administration`, `workflows`, or organization permissions for a contents-scoped dispatch.

The App owner records purpose, permission rationale, installation owner, selected repositories, key custodian, rotation deadline, and incident contact. A permission expansion is a reviewed security change.

> **Key insight**: A short-lived installation token is only narrow when both the registered permissions and the installation repository set are narrow; either one can widen the effective authority.

---

## 2. Keep the Private Key Out of General Repository Scope

Store the private key in an environment or external secret manager available only to the token-minting workflow. Mint a short-lived installation token for the named owner/repository, use it once, and never print it or pass it through job outputs.

```yaml
- id: app-token
  uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
  with:
    client-id: ${{ vars.CICD_APP_CLIENT_ID }}
    private-key: ${{ secrets.CICD_APP_PRIVATE_KEY }}
    owner: acme
    repositories: platform-deployments
```

Audit logs should show `cicd-dispatch[bot]`, not an employee, as the actor. If every repository can read the private key, the selected installation no longer limits who can mint its authority.

---

## 3. Rotate, Revoke, and Uninstall With Observable Proof

Rotation is overlap without ambiguity: create a new key, update the secret manager, run positive/negative access tests, then revoke the old key. The old key must fail to mint a token after revocation; existing installation tokens expire on their bounded lifetime.

On compromise, revoke keys, suspend or uninstall the App, invalidate dependent workflows, inspect App-actor audit events, and rotate any downstream credential it could reach. Recovery creates a new key and re-runs both installed and out-of-scope repository tests.

⚠️ Deleting a secret from one repository does not revoke the App key stored elsewhere.

⚠️ Removing a repository from the installation blocks new tokens there, but independently issued downstream cloud credentials may need separate revocation.

Do not create an App when the built-in `GITHUB_TOKEN` stays within one repository and already has the necessary permission. The App is for a distinct, auditable cross-repository or organization identity.

---

**Next**: [Workflow and Runner Hardening](02_workflow_and_runner_hardening.md)
