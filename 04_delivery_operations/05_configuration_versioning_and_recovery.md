# Configuration Versioning and Recovery

> **Who this is for**: Teams that already promote artifact digests through environments and roll back using the decision tree and workflow in [Verification, Observability, and Rollback](04_verification_observability_and_rollback.md). Read that note, and [Environments and Artifact Promotion](01_environments_and_promotions.md), first.

## The short version

A rollback redeploys the previous artifact digest — the immutable content hash from [promotion's release manifest](01_environments_and_promotions.md#2-promotion-moves-an-immutable-identity) — health checks turn green, and the same incident returns twelve minutes later, because the mutable configuration store still holds whatever values the bad release left behind. A digest pins which bytes run; it says nothing about which **configuration** — the resolved runtime settings and feature-flag states those bytes actually read at startup and on each request — they were tested against. Give configuration the same version discipline a digest already has: a content-hash **configuration version** that travels in the release manifest, a validation gate before it can promote, a snapshot of what was actually resolved, and a restore step that pairs it back to the digest during rollback.

**What you need (4 things):**

1. A configuration bundle — one file holding the resolved values and flag states an environment will actually run with, not just which flags are defined.
2. A content-hash **configuration version** for that bundle, computed the same way an artifact digest is.
3. A validation step that checks required keys and environment policy before promotion, not after.
4. An append-only record pairing each deployed digest with the configuration version that shipped alongside it.

**The code:**

```bash
#!/usr/bin/env bash
set -euo pipefail

config_file="${1:?usage: version-and-validate-config.sh CONFIG_FILE ENVIRONMENT}"
environment="${2:?usage: version-and-validate-config.sh CONFIG_FILE ENVIRONMENT}"

# Canonicalize (sorted keys, no whitespace) before hashing -- the same
# content-addressing reasoning an artifact digest relies on.
config_version="sha256:$(jq -Sc . "$config_file" | sha256sum | cut -d' ' -f1)"

missing="$(jq -r '["log_level","rate_limit_per_minute","feature_flags"] - keys | .[]' "$config_file")"
[[ -z "$missing" ]] || { echo "FAIL: missing required key(s): $missing" >&2; exit 1; }

if [[ "$environment" == "production" && "$(jq -r .log_level "$config_file")" == "debug" ]]; then
  echo "FAIL: log_level=debug is not allowed in production" >&2
  exit 1
fi

echo "config_version=${config_version}"
echo "OK: schema and policy checks passed for ${environment}"
```

**Success signal:** `config_version=sha256:46ea6fe68d1f7f9113ea3c70b432774694ca469b68e8bb0f136d49e1d7e29275` followed by `OK: schema and policy checks passed for production`; a debug log level or a missing key exits non-zero before either environment is touched.

**Not handled yet:** [driving these checks from a policy file instead of hardcoding them](#3-validate-configuration-before-it-can-promote-not-after), [what the snapshot captures and where it lives](#4-snapshot-what-was-actually-resolved-not-which-flags-exist), [restoring the paired version during rollback, with a fail-closed check](#5-restoring-configuration-is-part-of-rollback-not-an-afterthought), and [what happens when a shared mutable store bypasses all of this](#6-the-same-few-mistakes-undo-all-of-this).

---

For the recovery decision tree and how a rollback resolves its target digest, see [§6](04_verification_observability_and_rollback.md#6-recovery-is-a-decision-tree-not-a-single-rollback-button) and [§7](04_verification_observability_and_rollback.md#7-a-rollback-that-trusts-its-own-input-can-redeploy-the-wrong-or-revoked-artifact) of that note. This note assumes both and shows only what changes when configuration has to come back with the binary.

---

## 1. Redeploying the Old Binary Does Not Undo a Config Change

Digest `sha256:4ae0...9c1d` is live in production, paired with a configuration bundle that sets `rate_limit_per_minute: 500` as a plain integer. Release `sha256:7bd2...e614` ships next: new code that understands per-tier rate limits, plus a matching configuration change that reshapes `rate_limit_per_minute` into `{"default": 500, "premium": 2000}` and adds a new required key, `payment_retry_budget_ms`, that only the new binary reads. Ten minutes after rollout, error rates spike — from an unrelated dependency timeout, not from the rate-limit change at all. On-call opens incident `INC-1042` and runs the rollback workflow from [§7 of the rollback note](04_verification_observability_and_rollback.md#7-a-rollback-that-trusts-its-own-input-can-redeploy-the-wrong-or-revoked-artifact), which correctly resolves `sha256:4ae0...9c1d` as the rollback target and redeploys it.

Health checks turn green — the old binary starts, passes readiness, serves traffic. Then it fails on the first request that reads `rate_limit_per_minute`: the config store still holds `sha256:7bd2...e614`'s shape, a nested object, and the old binary's parser expects a plain integer. The rollback didn't fail; it succeeded at exactly what it was built to do — restore a digest — and that was never the whole job. Nothing in the rollback workflow touched the configuration store, because nothing told it configuration was part of what changed.

```text
binary rollback alone:
    old binary  +  whatever configuration happens to be live
                    (never validated against this binary, never intentionally paired with it)

binary + configuration rollback:
    old binary  +  the configuration version this binary actually shipped and was tested with
```

The fix is not "be more careful during incidents." It's giving configuration the same three properties a digest already has, so the rollback workflow can resolve and restore both without anyone having to remember to ask: an identifier that names an exact set of resolved values, a validation step that runs before that identifier is ever allowed to reach production, and a record of which identifier paired with which digest at deploy time.

---

## 2. A Configuration Version Travels in the Same Manifest as the Digest

> **Core:** this is the mechanism every section after it builds on — a configuration version is a content hash of the resolved bundle, carried as one more field in the release manifest [§2 of the promotion note](01_environments_and_promotions.md#2-promotion-moves-an-immutable-identity) already defines for `artifact:` and `source:`.

```text
build
└── artifact digest sha256:4ae0...9c1d
        │
configure
└── configuration version sha256:46ea...9275
        │
        ├── validate: schema + environment policy (§3)
        ├── deploy: digest and configuration version together
        ├── snapshot: resolved values → deployment record (§4)
        └── rollback: resolve digest AND its paired configuration version (§5)
```

Extend the promotion-inputs example directly:

```yaml
artifact:
  image: ghcr.io/acme/orders
  digest: sha256:4ae0...9c1d
source:
  repository: acme/orders
  commit: 8f31c2a7d9...
  build_run_id: 8912345678
configuration:
  bundle: s3://acme-config-bundles/orders/production/2026-08-10T10-00-00Z.json
  version: sha256:46ea...9275
  schema_version: 3
```

`version` and `image`/`digest` follow the same reasoning: a hash proves integrity, not location. `bundle` is the address — where to fetch the actual bytes, the same role `image` plays for the artifact — and `version` is the content hash of those bytes once canonicalized, computed exactly the way the short version's script computes it. Two bundles with identical resolved values hash identically no matter how they're formatted or in what order their keys were written; a single differing value — including a value nested inside `feature_flags` — changes the hash. `schema_version` is a separate, small integer that names which required-keys contract (§3) the bundle was validated against, so a validator written for schema 3 can refuse a schema-2 bundle instead of silently checking the wrong rules.

Never put secret values inside the bundle itself. A configuration version is meant to be logged, diffed, and stored in a widely-readable deployment record; a secret manager reference (an ARN or parameter path) belongs in the bundle, the secret's actual value does not — the same boundary [Permissions, Secrets, and OIDC](../03_security_and_supply_chain/01_permissions_secrets_and_oidc.md) draws for keeping credentials out of anything checked in or logged.

---

## 3. Validate Configuration Before It Can Promote, Not After

Say staging's bundle sets `internal_only_bypass_auth: true` so internal testers can skip the login flow during manual QA. Someone promotes a release by copying that staging bundle, changing the rate limit and a couple of hostnames for production, and forgetting the one boolean that mattered. The bundle is syntactically valid JSON, every required key is present, and the deploy succeeds — nothing about "valid configuration" caught that this specific value should never be allowed to leave staging. Every request to production now skips authentication, because the same code path reads that flag in both environments; the flag was only ever meant to gate a manual-testing convenience, and nothing enforced that boundary.

> **Production:** the short version's script hardcodes its required keys and its one policy check. That's fine for one environment and one rule. The moment a second environment or a second rule shows up, hardcoding turns into an ever-growing pile of `if` statements nobody wants to touch under deadline pressure — which is exactly when a rule gets skipped "just this once."

Move both the required-keys list and the disallowed-value rules into a policy file the script reads, so adding a rule is a data change instead of a script edit:

```json
{
  "required_keys": ["log_level", "rate_limit_per_minute", "feature_flags"],
  "disallowed": [
    { "check": ".log_level == \"debug\"", "message": "log_level=debug is not allowed in production" },
    { "check": ".feature_flags.internal_only_bypass_auth == true", "message": "internal_only_bypass_auth must not reach production" }
  ]
}
```

```bash
#!/usr/bin/env bash
set -euo pipefail

config_file="${1:?usage: validate-configuration.sh CONFIG_FILE POLICY_FILE}"
policy_file="${2:?usage: validate-configuration.sh CONFIG_FILE POLICY_FILE}"

config_version="sha256:$(jq -Sc . "$config_file" | sha256sum | cut -d' ' -f1)"

# Required keys come from the policy file rather than a hardcoded list --
# a new environment's requirements become a data change, not a script edit.
while IFS= read -r key; do
  jq -e "has(\"$key\")" "$config_file" > /dev/null \
    || { echo "FAIL: required key '$key' missing from $config_file" >&2; exit 1; }
done < <(jq -r '.required_keys[]' "$policy_file")

# Disallowed-value policy: a debug/verbose setting or bypass flag meant only
# for staging must never be a value promotion allows into a stricter
# environment. The policy file is reviewed like code -- it is trusted input.
rule_count="$(jq '.disallowed | length' "$policy_file")"
for i in $(seq 0 $((rule_count - 1))); do
  check="$(jq -r ".disallowed[$i].check" "$policy_file")"
  message="$(jq -r ".disallowed[$i].message" "$policy_file")"
  if jq -e "$check" "$config_file" > /dev/null; then
    echo "FAIL: $message" >&2
    exit 1
  fi
done

echo "config_version=${config_version}"
echo "OK: schema and policy checks passed against $(basename "$policy_file")"
```

Run against the staging bundle with the bypass flag still set, checked against production's policy:

```text
$ ./scripts/validate-configuration.sh staging_bypass.json policy-production.json
FAIL: internal_only_bypass_auth must not reach production
```

Wire it in as its own gate, parallel to `validate-release` from [promotion's workflow](01_environments_and_promotions.md#3-one-validated-digest-deploys-to-both-staging-and-production):

```yaml
jobs:
  validate-release:
    # ... digest and provenance checks, unchanged (01_environments_and_promotions.md §3)

  validate-configuration:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4
      - name: Version and validate the configuration bundle
        run: |
          ./scripts/validate-configuration.sh \
            config/production.json \
            config/policy-production.json

  staging:
    needs: [validate-release, validate-configuration]
    # ... unchanged

  production:
    needs: [verify-staging, validate-configuration]
    # ... unchanged
```

`staging` and `production` both add `validate-configuration` to their `needs:` list. A promotion whose configuration bundle fails this gate never reaches either environment — the same "validate once, deploy the validated identity everywhere" shape the digest already follows.

---

## 4. Snapshot What Was Actually Resolved, Not Which Flags Exist

A flag-definition list answers "what flags could this service check" — it says nothing about what any of them evaluated to for the deployment that just went out. A percentage-rollout flag might be defined once and resolve differently in staging (100% of an internal cohort) than in production (5% of all traffic) at the exact moment a given digest went live. The snapshot [§8 of the rollback note](04_verification_observability_and_rollback.md#8-what-you-retain-per-deployment-is-what-rollback-can-later-prove) already lists as something to retain per deployment is this resolved shape, not the flag catalog:

```json
{
  "environment": "production",
  "digest": "sha256:4ae0...9c1d",
  "config_version": "sha256:46ea...9275",
  "bundle": "s3://acme-config-bundles/orders/production/2026-08-10T10-00-00Z.json",
  "resolved_at": "2026-08-10T10:00:00Z",
  "resolved_values": {
    "log_level": "info",
    "rate_limit_per_minute": 500,
    "feature_flags": {
      "new_checkout": true,
      "recommendation_v2": false
    }
  }
}
```

`resolved_values` is a copy of exactly what the earlier validation step already canonicalized and hashed — not a re-derivation, and not a pointer that could later resolve to something different. Append one line per deployment to the same append-only deployment record `resolve-rollback-target.sh` reads:

```bash
#!/usr/bin/env bash
set -euo pipefail

environment="${1:?usage: append-deployment-record.sh ENVIRONMENT DIGEST CONFIG_VERSION}"
digest="${2:?usage: append-deployment-record.sh ENVIRONMENT DIGEST CONFIG_VERSION}"
config_version="${3:?usage: append-deployment-record.sh ENVIRONMENT DIGEST CONFIG_VERSION}"
record_file="${DEPLOYMENT_RECORD:-deployment-record.jsonl}"

# Append-only: never edit or delete a prior line, only add the newest one --
# resolve-paired-configuration.sh (§5) depends on being able to trust every
# entry it has ever seen.
jq -nc --arg env "$environment" --arg digest "$digest" --arg cv "$config_version" \
  --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  '{environment:$env, digest:$digest, config_version:$cv, deployed_at:$ts}' \
  >> "$record_file"
```

> **Production:** never let `resolved_values` include a secret. If a bundle references a secret manager path, snapshot the path, not the value it resolves to at runtime — the append-only record is exactly the kind of broadly-readable, incident-response-friendly store that should never double as a secret log. This is the same rule §2 states for the bundle itself, extended to what gets copied out of it.

---

## 5. Restoring Configuration Is Part of Rollback, Not an Afterthought

[§7 of the rollback note](04_verification_observability_and_rollback.md#7-a-rollback-that-trusts-its-own-input-can-redeploy-the-wrong-or-revoked-artifact) resolves a target digest from the deployment record and refuses anything revoked, data-incompatible, or non-adjacent — but it stops there. Nothing in that workflow reaches into the configuration store. Add one step that resolves the paired configuration version the same distrustful way, and refuses to guess when no pairing was ever recorded:

```bash
#!/usr/bin/env bash
set -euo pipefail

environment="${1:?usage: resolve-paired-configuration.sh ENVIRONMENT DIGEST}"
digest="${2:?usage: resolve-paired-configuration.sh ENVIRONMENT DIGEST}"
record_file="${DEPLOYMENT_RECORD:-deployment-record.jsonl}"

# The deployment record is append-only (§4), so the last matching line is
# the pairing that was actually live -- never derive a "closest" or
# "current" value when the exact pairing is missing.
match="$(jq -r --arg env "$environment" --arg digest "$digest" \
  'select(.environment == $env and .digest == $digest) | .config_version' \
  "$record_file" | tail -n1)"

if [[ -z "$match" ]]; then
  echo "FAIL: no configuration version is paired with ${digest} in ${environment} -- refusing to guess" >&2
  exit 1
fi

echo "$match"
```

Insert it into the rollback workflow between resolving the digest and deploying it — restoring configuration *before* the old binary starts, not after:

```yaml
      - name: Resolve rollback target from the deployment record
        id: resolve
        # ... unchanged (04_verification_observability_and_rollback.md §7)

      - name: Resolve paired configuration version
        id: resolve_config
        env:
          DIGEST: ${{ steps.resolve.outputs.digest }}
        run: |
          set -euo pipefail
          config_version="$(./scripts/resolve-paired-configuration.sh production "$DIGEST")"
          echo "config_version=$config_version" >> "$GITHUB_OUTPUT"

      - name: Restore paired configuration
        env:
          CONFIG_VERSION: ${{ steps.resolve_config.outputs.config_version }}
        run: ./scripts/apply-configuration.sh production "$CONFIG_VERSION"

      - name: Deploy previous digest
        env:
          DIGEST: ${{ steps.resolve.outputs.digest }}
        run: ./scripts/deploy-production.sh "ghcr.io/acme/orders@$DIGEST"
```

Configuration goes first. If it went last, the old binary would boot and briefly serve traffic against whatever configuration the *failed* release left behind — the exact gap §1's scenario shows. Restoring configuration first means the worst case is the still-running new binary reading old configuration for a few seconds, which it was already about to be replaced anyway; the reverse order means the *reintroduced* old binary runs against untested configuration, which is the failure this whole mechanism exists to close.

`resolve-paired-configuration.sh` failing is not an error to work around — it's the workflow correctly refusing to redeploy a combination nobody ever ran in production, exactly as `resolve-rollback-target.sh` refuses a revoked or non-adjacent digest. An operator hitting this mid-incident has one honest option: treat the missing pairing as its own incident finding, not a step to bypass by manually picking "whatever config looks current."

> **Key insight**: A digest is only half of "what was running." The other half is a value, not a fact about the binary — and unlike the binary, it can keep changing after the deployment that shipped it. Treating a digest and its configuration version as independently restorable is what lets "rollback" redeploy a combination that was never actually running in production.

---

## 6. The Same Few Mistakes Undo All of This

**⚠️ Configuration lives in a mutable store the deployment record never sees**

A dashboard, a `kubectl edit configmap`, or a direct write to a parameter store bypasses versioning entirely. The pairing step in §5 has nothing trustworthy to resolve — either it fails closed (correct, but now every rollback needs a manual override) or nothing enforces pairing at all, and the store just holds whatever was last touched. Route every configuration change through the same pipeline the bundle's version comes from; a change that isn't versioned isn't restorable.

**⚠️ The deployment record snapshots secret values, not references to them**

The moment someone greps the deployment record during an incident, every credential a bundle ever referenced is sitting in plaintext in a store built to be widely readable. This is the specific case §4's production note exists to prevent.

**⚠️ The restore step has no fail-closed path**

A script that falls back to "current" or "latest" configuration when no pairing is recorded reintroduces exactly the untested combination §5 exists to avoid — silently, because the workflow still reports success.

**⚠️ Configuration is validated once at promotion, then edited directly afterward**

The versioned, validated bundle and what's actually running diverge the moment anyone edits the live store out of band. The next rollback resolves a configuration version that matches the deployment record, not the drifted value actually in effect — restoring a state that was correct on paper and never correct in production.

---

## 7. When This Machinery Costs More Than the Problem

> **Edge case:** skip everything above when configuration is baked into the image at build time, with no separate deploy-time file at all. The artifact digest already pins it — versioning it a second time adds ceremony without adding any guarantee the digest didn't already provide.

If a managed platform already versions and validates configuration for you, don't re-implement hashing on top of it. AWS **AppConfig** — a managed configuration and feature-flag service — assigns each configuration profile its own version number and can run a JSON Schema or AWS Lambda validator against a version before it's ever deployed, rejecting one that fails ([Understanding validators](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-creating-configuration-and-profile-validators.html), checked 2026-08-14). In that setup, `configuration.version` in the manifest (§2) is that platform's own version identifier, not a hash you compute — the pairing and restore mechanism in §4–§5 stays the same, it just resolves and restores through that platform's API instead of a custom bundle file.

Also skip this for a single-service, single-environment project with one boolean flag and no compliance requirement to prove what was live when. The failure mode this note prevents — an incident that recurs because configuration didn't roll back with the binary — needs at least two environments and a value that's genuinely allowed to differ between them before it can happen at all.

---

## 8. Prove the Chain End to End

Version and validate a bundle, record the pairing it ships with, then resolve that pairing twice — once for a digest that has one, once for a digest that doesn't:

```bash
$ ./scripts/version-and-validate-config.sh production.json production
config_version=sha256:46ea6fe68d1f7f9113ea3c70b432774694ca469b68e8bb0f136d49e1d7e29275
OK: schema and policy checks passed for production

$ ./scripts/append-deployment-record.sh production "sha256:4ae0...9c1d" "sha256:46ea...9275"
$ cat deployment-record.jsonl
{"environment":"production","digest":"sha256:4ae0...9c1d","config_version":"sha256:46ea...9275","deployed_at":"2026-08-14T14:34:02Z"}

$ ./scripts/resolve-paired-configuration.sh production sha256:4ae0...9c1d
sha256:46ea...9275

$ ./scripts/resolve-paired-configuration.sh production sha256:0f3a...77b1
FAIL: no configuration version is paired with sha256:0f3a...77b1 in production -- refusing to guess
```

**Success signal:** the first resolve prints exactly the version just recorded — proving the pairing round-trips — and the second exits non-zero with the fail-closed message instead of printing anything a caller could mistake for a valid version. (The pipeline stores the full 64-character hash the first command prints; the deployment record and later commands use the same abbreviated `sha256:46ea...9275` form this collection uses for digests, pointing at the identical committed bundle.)

That last line is the whole mechanism working as designed: a digest with no recorded configuration pairing — predating this retention, or deployed by some path that skipped it — cannot be rolled back to silently. Something upstream of this script has to go find out what configuration actually ran, before rollback is allowed to touch it.

---

## 9. References

- [AWS AppConfig — Understanding validators](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-creating-configuration-and-profile-validators.html) (checked 2026-08-14)
- [AWS AppConfig — Creating configuration profiles](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-creating-configuration-and-profile.html) (checked 2026-08-14)
- [The Twelve-Factor App — Config](https://12factor.net/config)

---

**Next**: [Provisioning the Reference Platform](../05_reference/01_provisioning_the_reference_platform.md)
