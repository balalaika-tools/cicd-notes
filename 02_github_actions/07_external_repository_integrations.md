# Reverse-Engineering External Repository Integrations

> **Who this is for**: Engineers inheriting a repository whose checks, comments, or deployments are produced by a GitHub App or service configured outside Git.

## A Repository Can Show the Input Without Showing the Automation

A merge to `deploy/orders/production.yaml` changes production, yet no workflow mentions that path. The missing mechanism is an **external control plane**: an administrator configured a GitHub App or service to watch repository input and reconcile it elsewhere. Treat the repository, integration settings, and runtime as three separate evidence surfaces.

```text
deploy/orders/production.yaml changed to digest sha256:4ae0...
  → check "Acme Deploy / validate" appears from acme-deploy[bot]
  → merge to configured publication branch
  → external service reconciles runtime
  → platform reports sha256:4ae0... serving
```

The check proves detection, not deployment. The runtime observation closes the loop.

---

## 1. Start With Observable Evidence and Preserve Unknowns

> **Core:** keep repository intent, external binding, and runtime effect as separate rows; this evidence boundary is the reusable mechanism.

Inventory repository artifacts first: watched-looking folders, App-authored checks/comments, webhook deliveries visible to administrators, deployment statuses, and bot pull requests. Then record the external owner and authoritative inspection surface.

```yaml
connection: acme-deploy-production
owner: platform-delivery
source:
  repository: acme/deployments
  branch: main
  path: deploy/orders/production.yaml
mapping:
  service_key: orders
  runtime_target: ecs/production/orders
detection_evidence: "check: Acme Deploy / validate"
publication_boundary: "merge to main"
runtime_verification: "ECS task image digest + /version digest"
unknown_until_admin_verified:
  - webhook_or_polling
  - retry_schedule
```

Do not infer webhook versus polling, watched branch, or target mapping from repository layout. An absent workflow proves only that the workflow is absent.

> **Key insight**: Repository evidence proves source intent; external settings prove the binding; runtime evidence proves effect. None substitutes for another.

---

## 2. Prove Detection, Publication, and Consumption Separately

Use one changed-input contrast:

| Change | Expected detection | Expected runtime result |
|---|---|---|
| allowed path on `main`: digest A → B | App check names B and passes | target converges to B |
| same file on unconfigured branch | no publication/reconcile | target remains B |

The administrator exports the connection settings before the test. After merge, query the platform's resolved digest and the service's reported digest. If either surface is inaccessible, record `unknown`; do not turn a green App check into deployment success.

---

## 3. Drift and Failure Have Distinct Tells

⚠️ A check never appears: the App may be uninstalled, its repository scope may exclude this repository, or the watched path/branch may differ.

⚠️ A check passes but runtime stays old: detection succeeded; reconciliation, identifier mapping, or target credentials failed.

⚠️ Runtime changes without a matching merge: another writer or control plane can mutate the target; investigate before reconciling.

Do not use this pattern when repository-owned Actions can express the whole mechanism and no independent service is required. Prefer visible, versioned automation when the external control plane adds no capability.

---

**Next**: [Cross-Repository Contract Lifecycle](08_cross_repository_contract_lifecycle.md)
