# Cross-Repository Contract Lifecycle

> **Who this is for**: Producer and consumer teams sharing a library, schema, action, module, or chart across repositories.

## Publish an Immutable Version Before Asking Consumers to Move

`orders-contract` publishes `2.4.1` with digest `sha256:91ab...`; `billing` pins both, an update bot proposes `2.5.0`, and compatibility tests pass before merge. That is a contract lifecycle. Copying `main` is not: the consumer cannot name, test, or roll back what changed.

```text
producer release 2.4.1 → immutable registry digest sha256:91ab...
consumer lock          → version 2.4.1 + sha256:91ab...
update PR              → 2.5.0 + compatibility result
deprecation            → 2.x support deadline; 3.0 removes old shape
```

---

## 1. Producer Metadata Makes Compatibility Reviewable

> **Core:** the producer publishes immutable metadata and a compatibility promise before any consumer update is proposed.

The producer owns publication and the compatibility promise:

```yaml
name: orders-contract
version: 2.5.0
digest: sha256:c312...
compatibility:
  previous_supported: ">=2.4.0 <3.0.0"
  breaking_changes: []
schema: openapi/orders.yaml
source: acme/orders-contract@8f31c2a
```

Publishing succeeds only after producer tests validate old and new fixtures. The registry must refuse overwriting `2.5.0`; a second publication with different bytes is a failure.

> **Key insight**: Versioning is useful only when an immutable publication, a compatibility promise, and a consumer verification all name the same bytes.

---

## 2. The Consumer Pins Identity and Tests Its Own Assumptions

The consumer owns adoption:

```toml
[dependencies]
orders-contract = "2.5.0"

[integrity]
orders-contract = "sha256:c312..."
```

The update bot opens a PR changing both values. CI validates registry digest, generates the client, runs provider fixtures against the consumer, and exercises one old and one new payload. Success is a green compatibility check on the exact PR commit; a version/digest mismatch fails before tests.

---

## 3. Deprecation Is a Measured Transition, Not a Changelog Line

Additive `2.5.0` keeps the old field and adds an optional field with a default. Old and new consumers pass. Producers measure remaining old-field use, notify owners, and publish a deadline. Only `3.0.0` removes it after supported consumers migrate.

```text
2.5.0 publish → canary consumers → updater PRs → usage reaches zero
             → deprecation deadline → 3.0.0 breaking release
```

⚠️ A producer removes a field in `2.5.1`: the version claims compatibility while the schema breaks it.

⚠️ An update PR is green without producer/consumer fixtures: it proves installation, not compatibility.

Skip a registry contract for source that is always released and deployed atomically in one repository. Use it when producer and consumer can move independently.

---

**Next**: [Performance and Reliability](06_performance_and_reliability.md)
