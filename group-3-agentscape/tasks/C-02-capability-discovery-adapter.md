# C-02 — Capability Discovery Adapter

## 1. Task Identity

```text
Task ID:        C-02
Parent Plan:    AS-02 / Gate L2
Owner Track:    Provider / Connector
Status:         MERGED
Base Commit:    fefa495
Final Commit:   47470c0
Feature Branch: feat/c02-capability-discovery
```

## 2. Goal

Build a versioned Connector capability-snapshot consumer that converts `/connector/v1/capabilities` into the canonical AgentScape `ProviderRegistry` view **without exposing provider-private transport schemas, without mutating Runtime truth, and without allowing a Connector snapshot to overwrite locally owned providers**.

## 3. Non-goals

- no Async Job state machine;
- no Job DB or Job Center UI;
- no Artifact importer;
- no real Modal GPU submission;
- no provider execution binding yet;
- no Physics/Interaction/WorldRuntime changes;
- no automatic promotion of provider semantics into `manifest.actions` or Runtime capability;
- no credential management.

## 4. Preconditions

- C-01 commit `dbb52a5` exists and is clean;
- Connector authenticated request boundary supports `capabilities.read`;
- P-01 ProviderRegistry is already merged in main;
- current main dirty EmbodiedGen work is treated as a separate ownership track;
- C-02 implementation remains in an isolated worktree/branch until C-01 is merged.

## 5. Current Code Facts

`ProviderRegistry` currently contains static provider descriptors and rejects duplicate provider IDs. The default registry already includes disabled placeholders for `modal-2d` and `modal-3d`.

That means C-02 cannot safely implement discovery as:

```text
GET capabilities
  -> for each provider
     registerProvider(provider)
```

because a legitimate dynamic snapshot for `modal-2d` would collide with the existing placeholder. Likewise a blind `replaceProvider(id)` would allow a Connector snapshot to overwrite locally-owned providers such as `local-catalog` or `legacy-http-generator`.

## 6. Required capability envelope

C-02 should normalize a response shaped conceptually like:

```text
CapabilitySnapshot
├── contractVersion
├── connector identity/instance/version
├── revision
├── hash
├── generatedAt
├── expiresAt/cache policy
└── providers[]
    ├── provider id/display/version
    ├── implementationRevision
    ├── health/status
    ├── supported Connector range
    └── capabilities[]
        ├── stable operation ID
        ├── input schema/limits
        ├── profiles/options schema
        ├── output roles
        ├── execution async/stages/cost/duration
        ├── prerequisites
        ├── cancel/resume/idempotency
        └── consumer hints/warnings/deprecation
```

Unknown optional fields may be ignored or retained in raw provenance; unknown major contract versions must fail closed.

## 7. Registry ownership model

C-02 must introduce explicit provider ownership/provenance rather than generic replacement.

Recommended model:

```text
ProviderRegistry
├── local owned providers
│   ├── local-catalog
│   └── legacy-http-generator
│
├── static compatibility placeholders
│   ├── modal-2d disabled
│   └── modal-3d disabled
│
└── connector snapshot ownership
    ├── sourceKind=connector
    ├── connectorInstance
    ├── capabilityRevision
    ├── capabilityHash
    └── provider IDs owned by this snapshot
```

An atomic snapshot apply may update a placeholder only when that provider is declared connector-owned/replaceable. It must not overwrite arbitrary local providers.

A later snapshot from the same connector may replace only providers previously owned by that same connector snapshot. Providers removed from the new snapshot should become unavailable/removed according to a deterministic rule; stale providers must not silently remain `available`.

## 8. Suggested public interfaces

Keep the first interface narrow:

```text
ConnectorCapabilityAdapter
  normalizeSnapshot(payload, sessionDescriptor)
  fetchSnapshot(connectorClient)

ProviderRegistry
  applyProviderSnapshot(snapshot, ownership)
  getProviderSource(id)
  getSnapshotState(sourceId)
```

Exact names can change, but the semantics must include atomic validation-before-commit.

## 9. Atomic update requirement

Snapshot application must be all-or-nothing:

```text
receive snapshot
  |
validate envelope/version/hash linkage
  |
normalize every provider/capability
  |
check ownership conflicts
  |
ALL valid ?
  |      |
 no      yes
  |       |
 reject   atomic registry swap
  |
existing registry unchanged
```

A malformed second provider must not leave the first provider partially updated.

## 10. Session binding requirement

The capability snapshot must match the active C-01 session:

```text
session.capabilityRevision == snapshot.revision
session.capabilityHash     == snapshot.hash
connector instance         == snapshot.connector.instance
contract major             == supported major
```

Mismatch is stale/untrusted discovery evidence and must fail closed.

## 11. Execution boundary

C-02 is discovery only. Dynamic capabilities should not automatically become executable.

Current `ProviderRegistry.resolveCapability()` defaults to `executableOnly=true`; preserve that property. Connector-discovered capability descriptors may be queryable for UI/strategy planning, but Job execution bindings belong to J-01/J-02.

This preserves:

```text
Capability advertised
      !=
Executable binding
      !=
Provider job succeeded
      !=
Asset ready
```

## 12. Security / Trust Boundary

C-02 must reject or strip:

- secrets/tokens/Authorization-like fields;
- arbitrary filesystem/Volume paths as public capability identity;
- provider-private function/App names as stable operation IDs;
- operation IDs outside `<provider>.<domain>.<operation>.v<major>`;
- unknown auth modes;
- capabilities claiming credential-management scopes;
- a Connector attempt to overwrite local-owned provider IDs.

Provider semantic evidence must remain metadata only; it must not add Runtime actions.

## 13. Required tests

At minimum:

```text
valid modal-2d/modal-3d snapshot normalizes
session revision/hash match
revision mismatch rejected
hash mismatch rejected
connector instance mismatch rejected
unknown major contract rejected
malformed second provider -> no partial registry mutation
connector placeholder upgrade allowed
local-catalog overwrite rejected
legacy-http-generator overwrite rejected
removed connector provider no longer remains falsely available
unknown optional fields tolerated
secret-like fields not exposed in normalized descriptor
new capability remains non-executable without binding
```

## 14. Regression requirements

Must retain:

```text
connector-session tests
provider-registry tests
asset-library tests
world-generation-pipeline tests
```

No changes to Physics/Interaction tests should be required.

## 15. Definition of Done

- [ ] capability envelope normalized;
- [ ] session revision/hash/connector binding enforced;
- [ ] registry ownership model implemented;
- [ ] atomic snapshot apply implemented;
- [ ] local provider overwrite blocked;
- [ ] stale provider behavior deterministic;
- [ ] dynamic capabilities remain non-executable;
- [ ] focused tests PASS;
- [ ] full JS regression PASS;
- [ ] production build PASS;
- [ ] feature commit pushed;
- [ ] C-01 merge state rechecked before merge.

## 16. Completion Evidence

```text
Final Status:       MERGED
Final Commit:       47470c0 feat: add connector capability snapshots
Merged into main:   yes
CodeGraph impact:   applyProviderSnapshot = 5 nodes / 4 edges
Focused tests:      4 files / 33 tests PASS
Full JS regression: 116 files / 424 tests PASS
Asset validation:   PASS
Production build:   PASS (exit_code=0, 19.05s)
Python smoke:       4/4 PASS
GitHub Actions:     build success / deploy success
CI run:             32698268133
```

C-02 preserves the key boundary: discovered capability descriptors are queryable but remain non-executable until a later Job execution binding exists.

### Unlocked Gate

C-01 + C-02 satisfy the Connector session/capability foundation required to begin J-01 Async Generation Job projection.
