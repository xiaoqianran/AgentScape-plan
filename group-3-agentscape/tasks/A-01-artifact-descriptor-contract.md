# A-01 — Artifact Descriptor / Identity / Lineage Contract

## 1. Task Identity

```text
Task ID:        A-01
Parent Plan:    AS-04 / Gate L4
Owner Track:    Artifact
Status:         READY
Base Commit:    140acd9
Feature Branch: feat/a01-artifact-descriptor
```

## 2. Goal

Introduce AgentScape's canonical Artifact domain contract before any bytes importer exists: opaque artifact identity, immutable content fingerprint, safe location metadata, producer/derivation lineage, retention/lease semantics, and deterministic descriptor validation.

A-01 does **not** download artifact bytes. It establishes the truth model that A-02 will enforce while streaming bytes.

## 3. Core invariants

```text
Artifact ID          != path
Artifact ID          != signed URL
Artifact ID          != SHA-256
SHA-256              = immutable content fingerprint
Location             = mutable availability/access metadata
Deleting a location  != deleting artifact lineage
Provider Job success != Artifact bytes verified
Artifact verified    != Asset compiled/admitted
```

## 4. Canonical Artifact Descriptor

Suggested shape:

```text
ArtifactDescriptor
├── id                  opaque URL-safe identity
├── role                primary-glb / manifest-json / urdf / preview / ...
├── type/schema         stable semantic/schema identity
├── displayName         safe UI label, never path
├── mime
├── format
├── bytes               declared exact byte length
├── hash                 canonical sha256:<64 lowercase hex>
├── producer
│   ├── jobId
│   ├── provider
│   ├── operation
│   ├── stage
│   ├── attempt
│   └── revision/model/workflow safe refs
├── lineage
│   └── parents[] { artifactId, hash, relation }
├── createdAt
├── expiresAt           artifact metadata expiry, optional
├── integrity           declared / verified / rejected
├── warnings[]          safe strings/codes
├── retention
└── locations[]
```

## 5. Hash contract

A-01 accepts only canonical SHA-256 identity:

```text
sha256:<64 lowercase hex>
```

No base64 ambiguity, no weak hashes, no user label derived from hash.

A-02 will compute bytes/hash; A-01 only validates declared descriptor shape.

## 6. Location contract

AgentScape must not ingest raw provider Volume paths or signed URLs as stable descriptor fields.

Canonical browser-safe location metadata should include only:

```text
id
kind                connector | local-cache | compiled-store | legacy
scope               session | job | project | application
state               available | unavailable | expired | evicted
verifiedAt
expiresAt
access               safe transport descriptor, if applicable
```

For Connector bytes, preferred access is:

```text
kind = connector-artifact
artifactId = <same opaque artifact ID>
```

The actual HTTP route is derived by trusted Connector client code, not copied from provider response.

Reject/strip:

```text
path
volumePath
signedUrl
url with token/query
Authorization
provider secret
```

## 7. Producer / lineage

Producer identity should be safe/stable:

```text
jobId
provider
operation
stage
attempt
revision
model {id/version/revision}
workflow {id/version/revision}
```

Do not expose provider private FunctionCall ID as artifact identity.

Derivation lineage:

```text
parents[]
  artifactId
  hash
  relation = derived_from | input | preview_of | compiled_from
```

Lineage is immutable provenance. Location eviction must not remove it.

## 8. Integrity/readiness semantics

A-01 must avoid calling declared metadata “verified”.

Recommended states:

```text
declared   descriptor accepted but bytes not independently checked
verified   A-02/importer has checked bytes + hash + MIME/structure gate
rejected   integrity gate failed
```

A-01-created descriptors default to `declared` unless a trusted importer explicitly provides verified evidence through a separate transition API.

`verified` still does not mean AgentScape Asset Admission ready.

## 9. Retention and lease

Retention policy is metadata, separate from remote TTL/location expiry.

Suggested classes:

```text
ephemeral
session
project
favorite
persistent
```

Lease contract:

```text
lease ID
artifact ID
holder kind/id
createdAt
expiresAt optional
reason
```

Active lease prevents cleanup of the relevant local/transfer location, but does not mutate artifact content identity.

A-01 may implement an in-memory ArtifactRegistry/lease index; durable persistence is later work.

## 10. Registry semantics

Recommended `ArtifactRegistry` responsibilities:

- register normalized descriptor;
- reject same artifact ID with conflicting immutable content identity;
- allow safe location state updates without changing hash/bytes/lineage;
- retain descriptor after location removal;
- index by content hash for later A-02 dedupe lookup;
- manage in-memory leases;
- expose safe snapshots only.

Do not silently merge two different artifact IDs solely because hashes match. Hash index may report equivalent content; canonical identity decisions belong to Connector/project policy.

## 11. Security boundary

Descriptor normalizer must reject secret-like fields recursively in untrusted surfaces.

No:

- Authorization/API key/token/credential;
- signed URL;
- raw provider/local filesystem path;
- prompt full text;
- image bytes/base64 payload;
- remote traceback;
- provider private call identity.

Safe display names/warnings should have size limits and reject control characters.

## 12. Required tests

At minimum:

```text
valid descriptor normalizes
invalid/weak/noncanonical hash rejected
negative/unsafe bytes rejected
unsafe artifact ID rejected
safe displayName not used as path
secret/path/signed-url fields rejected
producer Job ID/operation validated
parent lineage normalized
unknown lineage relation rejected
location path/url rejected
location expiry/state normalized
same artifact ID + same immutable identity idempotent
same artifact ID + different hash/bytes rejected
remove location retains descriptor/lineage
hash index returns content-equivalent artifact IDs
lease acquire/release
active lease reported independently of location state
declared != verified
verified transition requires explicit integrity evidence API
```

## 13. Non-goals

- no fetch/download;
- no Range resume;
- no MIME magic sniff;
- no GLB parse;
- no hash computation over bytes;
- no IndexedDB/cache persistence;
- no Compiler call;
- no Asset Admission;
- no World mutation.

Those belong to A-02 and later gates.

## 14. Merge Gate

- Artifact modules only; no Physics/Interaction/WorldRuntime changes;
- full Job/Connector regression stays green;
- full JS regression PASS;
- production build PASS;
- secret scan clean;
- CodeGraph impact remains isolated;
- CI/Pages deploy PASS.
