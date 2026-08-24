# A-02 — Streaming Artifact Importer / Bytes + Hash + MIME Gate

## 1. Task Identity

```text
Task ID:        A-02
Parent Plan:    AS-04 / Gate L4
Owner Track:    Artifact / Connector
Status:         MERGED
Base Commit:    ac3f996
Final Commit:   09e63cc
Feature Branch: feat/a02-artifact-importer
```

## 2. Goal

Import bytes for an A-01 declared Artifact through the trusted Connector artifact facade using bounded streaming, incremental SHA-256, exact length checks, MIME/format sniffing, temporary storage with abort/commit semantics, and explicit integrity verification.

A-02 completes **Artifact integrity**, not Asset Admission.

## 3. Truth boundary

```text
Job succeeded
   |
   v
Artifact descriptor declared
   |
   v
A-02 stream bytes
   |
   +-- length gate
   +-- maxBytes budget
   +-- incremental SHA-256
   +-- Content-Type gate
   +-- magic/format structure gate
   +-- temporary store commit
   |
   v
Artifact integrity = verified
   |
   X
NOT Asset ready / compiled / admitted
```

## 4. Connector transport

Trusted route is derived from opaque artifact ID:

```text
GET /connector/v1/artifacts/{artifactId}
scope = artifacts.read
```

No signed URL or provider path is accepted from ArtifactDescriptor.

Requirements:

- path-safe opaque ID only;
- Connector session Authorization stays inside C-01 client boundary;
- redirects are rejected/fail closed;
- `Content-Length` is checked when present;
- `Content-Type` must be compatible with descriptor MIME;
- missing `Content-Length` is allowed only under streaming byte budget;
- non-2xx response maps to structured artifact transport error.

## 5. Streaming SHA-256

Current browser dependency set has no incremental hash library. `crypto.subtle.digest()` is whole-buffer only and must not be presented as streaming.

A-02 should add a small deterministic incremental SHA-256 implementation with:

```text
update(Uint8Array)
digestHex()
digestArtifactHash() -> sha256:<hex>
```

Tests compare random/chunked vectors against Node `crypto.createHash('sha256')` and standard known vectors.

## 6. Byte budget

Importer receives a configurable `maxBytes` hard limit. Before and during streaming:

```text
Content-Length > descriptor.bytes -> reject
Content-Length != descriptor.bytes -> reject
Content-Length > maxBytes          -> reject
streamed bytes > maxBytes          -> abort immediately
streamed bytes > descriptor.bytes  -> abort immediately
EOF bytes != descriptor.bytes      -> reject truncated/length mismatch
```

No unbounded `arrayBuffer()` of the network response.

## 7. Temporary byte store

Define a narrow byte-store transaction API so current tests can use memory and later persistence can use IndexedDB/cache without changing importer truth:

```text
begin(artifact)
  -> writer.write(chunk)
  -> writer.commit({hash,mime,bytes})
  -> writer.abort()

store.get(key)
store.remove(key)
store.findByHash(hash) optional
```

Rules:

- bytes are not published to a stable local-cache location before all integrity gates pass;
- any fetch/hash/MIME/structure error aborts temp writer;
- if a failure occurs after commit but before Registry finalize, committed cache entry is removed;
- temporary object URLs are not created by core importer;
- byte-store internals are not exposed as raw filesystem paths.

## 8. Transfer lease

Importer acquires an artifact-wide `transfer` lease before streaming and releases it in `finally`.

The lease protects source/local cleanup during transfer but does not mutate Artifact identity.

Lease ID must be locally generated opaque identity, injectable in tests.

## 9. MIME + structure gate v1

A-02 v1 detects content from bytes instead of trusting extension/provider field.

Required supported gates:

### GLB

```text
magic = glTF
version = 2
header totalLength == actual bytes
MIME = model/gltf-binary
```

Corrupt/truncated/mismatched-length GLB is rejected before `verified`.

### JSON

```text
UTF-8 decode
JSON.parse succeeds
MIME = application/json (or explicit approved +json subtype if contract adds one)
```

Parsed value is still untrusted semantic data; A-02 only proves byte/structure integrity.

### PNG/JPEG/WebP

Magic-signature detection is sufficient for MIME integrity in this slice; pixel/alpha/decode bounds belong image-specific contract work.

### XML/text/OBJ

May be supported by bounded UTF-8/text signature gates, but do not invent semantic validity.

### ZIP/archive

A-02 v1 fails closed on archive MIME/format until a bounded bundle parser validates entry count, decompressed bytes and traversal. Do not partially unzip.

## 10. Registry finalize order

Successful import should be transactional in this order:

```text
1. descriptor already registered as declared
2. acquire transfer lease
3. begin temp writer
4. stream + hash + bytes
5. MIME/structure validation
6. temp writer commit -> safe cache key
7. registry.updateLocation(local-cache available)
8. registry.verifyIntegrity(hash,bytes,mime,method)
9. release transfer lease
```

On failure:

- temp writer abort;
- committed entry removed if necessary;
- local-cache location removed if added and not protected by another lease;
- transfer lease released;
- descriptor/lineage retained;
- network interruption does **not** automatically mark global integrity rejected;
- actual hash/length/MIME/structure contradiction may record rejection evidence if policy chooses, but must not delete identity.

## 11. Deduplication boundary

A-02 may expose/store content by SHA-256 and detect an already verified local byte entry.

Do not yet claim “compiled asset reuse” from hash alone. Existing compiled Asset lookup/admission integration belongs a later AS-07/Compiler bridge.

## 12. Range resume

Connector range resume is optional in AS-04 wording. A-02 first merge may deliberately omit it if the stream/abort contract is complete.

If later added:

- resume only from verified partial length;
- `206 Content-Range` must match expected artifact ID/total bytes;
- final SHA-256 always covers entire artifact;
- never trust a resumed segment hash alone.

## 13. Required tests

At minimum:

```text
incremental SHA-256 known vectors
chunked SHA-256 == Node crypto
valid streamed GLB -> verified
valid JSON -> verified
Content-Length exact check
missing Content-Length bounded stream works
oversize header rejected before read
stream exceeds maxBytes -> abort
stream exceeds descriptor.bytes -> abort
truncated stream rejected
hash mismatch rejected
Content-Type mismatch rejected
GLB bad magic/version/header length rejected
malformed JSON rejected
temp writer aborted on network reader failure
committed bytes cleaned if Registry finalize fails
transfer lease always released
source descriptor/lineage retained on failure
Connector path derived only from artifact ID
artifacts.read scope enforced
redirect/non-2xx rejected
archive MIME fails closed
verified artifact still != Asset Admission ready
```

## 14. Non-goals

- no Range resume in first slice;
- no ZIP extraction;
- no URDF semantic parser;
- no GLTF semantic compile;
- no AssetCompiler call;
- no Asset Admission;
- no IndexedDB durable implementation;
- no World mutation.

## 15. Merge Gate

- A-01 regression green;
- J/C/P regressions green;
- full JS regression PASS;
- production build PASS;
- Python smoke PASS;
- CodeGraph impact remains Artifact/Connector-only;
- GitHub Actions build + Pages deploy PASS.

## 16. Completion Evidence

```text
Final Status:       MERGED
Final Commit:       09e63cc feat: add streaming artifact integrity importer
Merged into main:   yes
Focused A-02:       9 files / 69 tests PASS
Wide regression:    18 files / 133 tests PASS
Full JS regression: 132 files / 536 tests PASS
Asset validation:   PASS
Production build:   PASS (exit_code=0, 13.86s)
Python smoke:       4/4 PASS
GitHub Actions run: 32707156221
Test and build:     success
Pages deploy:       success
```

Additional correctness boundaries implemented:

- incremental SHA-256 is chunked and verified against Node crypto, not whole-buffer WebCrypto;
- Connector artifact source binds artifact ID to connector id + connector instance;
- authenticated artifact fetches use `artifacts.read`, `credentials=omit`, and `redirect=error`;
- streaming has hard maxBytes/maxChunks limits;
- JSON has maxStructuredBytes/maxJsonDepth/maxJsonNodes;
- GLB validates v2 header, total length, JSON first chunk type/alignment/object start;
- archive MIME fails closed;
- temp bytes publish only after all integrity gates;
- failure cleanup retains descriptor/lineage and releases transfer lease;
- provider/network failure does not fabricate verified/rejected Artifact truth.

### Unlocked Gate

AS-05 Generic modal-3D single-asset production loop is ready: verified GLB Artifact -> AssetCompiler -> Asset Admission -> Manifest registration.
