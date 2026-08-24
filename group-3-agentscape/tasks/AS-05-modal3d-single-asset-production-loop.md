# AS-05 — Generic modal-3D Single Asset Production Loop

## 1. Task Identity

```text
Task ID:        AS-05
Parent Plan:    AgentScape Integration Plan / Gate L5
Owner Track:    Artifact -> Compiler -> Asset Admission
Status:         MERGED
Base Commit:    09e63cc
Final Commit:   162a101
Feature Branch: feat/as05-modal3d-single-asset
```

## 2. Goal

Close the first real vertical production path from an already-verified `modal-3d.asset.image_to_3d.v1` Artifact into AgentScape's existing compiler/admission system:

```text
verified GLB Artifact
  -> local verified bytes
  -> AssetCompiler.compile({bytes, sourceName, assetId, label})
  -> compiler report / quality
  -> Asset Admission
  -> manifest registration
  -> asset becomes eligible for spawn / WorldSpec
```

This is the first point where remote generated content may become an AgentScape Asset. It must use existing Compiler/Admission truth rather than creating a parallel "provider asset" path.

## 3. Non-goals

- no modal-2D/SAM composition yet;
- no EmbodiedGen bundle changes;
- no World placement/composer changes;
- no automatic spawn;
- no bypass for provisional compiler quality;
- no signed URL/path based loading;
- no provider semantic evidence promoted directly into runtime actions;
- no retry/cost strategy orchestration.

## 4. Preconditions

- C-01/C-02 Connector session/capability merged;
- J-01/J-02 async Job truth/reconcile merged;
- A-01 Artifact identity/lineage merged;
- A-02 verified local bytes importer merged;
- Artifact integrity must be `verified` before compiler invocation;
- Artifact MIME/format must be `model/gltf-binary` / `glb` for this first slice.

## 5. Core readiness chain

Do not collapse these states:

```text
provider succeeded
!= artifact verified
!= compiler succeeded
!= asset admitted
!= manifest registered
!= world spawned
!= task verified
```

Recommended production result phases:

```text
artifact_verified
compiling
compiled
admitted | provisional | rejected
registered
```

A provisional compiler/admission result is a valid truth result, not a transport failure.

## 6. Byte ownership

AS-05 consumes bytes from the trusted A-02 local byte store using the verified local-cache location recorded in ArtifactRegistry.

It must not fetch Connector bytes again if verified local bytes are present.

Before compile:

```text
Artifact.integrity.state == verified
local-cache location == available
cache entry hash == Artifact.hash
cache entry bytes == Artifact.bytes
cache entry mime == Artifact.mime
```

Any mismatch fails closed before AssetCompiler.

## 7. Compiler contract

Use existing `AssetCompiler.compile` entry rather than new provider-specific compiler code.

Expected call shape:

```text
AssetCompiler.compile({
  bytes,
  sourceName,
  assetId,
  label,
  ...existing safe compile options only
})
```

Derive stable values from Artifact descriptor:

```text
assetId      = explicit AgentScape asset identity derived by caller/policy
sourceName   = safe display name or `${artifact.id}.glb`
label        = safe display label
```

Artifact ID and Asset ID remain distinct identities.

## 8. Admission contract

Compiler output must pass existing Asset Admission path.

No rule like:

```text
provider == modal-3d -> admitted
```

Generic modal-3D assets without URDF/affordance evidence may correctly remain provisional because fallback colliders/semantics are lower confidence.

AS-05 should preserve compiler/admission reasons and warnings in production result provenance.

## 9. Manifest registration

Manifest registration only happens after Admission says the asset is eligible according to current repository rules.

If repository distinguishes accepted/provisional registry states, preserve that distinction. Do not mutate Manifest actions to improve apparent success rate.

## 10. Provenance

Production result must retain safe links:

```text
sourceArtifactId
sourceArtifactHash
producerJobId
provider
operation
capabilityRevision/hash (if already available through Artifact producer lineage)
compiler report id/revision if existing
admission decision/reasons
assetId
manifest id/version
```

No token, provider remote call ID, cache internal bytes, or path.

## 11. Transaction boundary

Recommended orchestration module:

```text
src/pipeline/VerifiedArtifactAssetPipeline.js
```

Responsibilities:

1. validate verified Artifact + local cache identity;
2. acquire Artifact lease during compile/admission;
3. call existing AssetCompiler;
4. call existing Asset Admission;
5. register result through existing Asset Library/manifest mechanism;
6. return structured stage/provenance result;
7. always release lease.

Do not duplicate compiler or admission internals.

## 12. Failure semantics

Separate categories:

```text
ARTIFACT_NOT_VERIFIED
ARTIFACT_LOCAL_BYTES_UNAVAILABLE
ARTIFACT_CACHE_IDENTITY_MISMATCH
COMPILER_FAILED
COMPILER_QUALITY_REJECTED / provisional
ASSET_ADMISSION_REJECTED
MANIFEST_REGISTRATION_FAILED
```

A compiler exception must not change Artifact integrity to rejected: the bytes may be valid but unusable by current compiler policy.

A failed Asset Admission must not remove verified Artifact/cache; retry/recompile policy can reuse it.

## 13. Required tests

At minimum:

```text
verified modal-3d GLB -> existing AssetCompiler called with bytes
unverified Artifact rejected before compiler
no available local-cache location rejected
cache hash/length/MIME mismatch rejected before compiler
Connector source is not re-fetched
transfer/compile lease held during pipeline and released on success/failure
compiler exception -> Artifact remains verified/cache retained
compiler provisional result remains provisional
compiler reject does not fabricate admission success
admission reject retains compiler report + Artifact
manifest registration only after admission eligibility
successful path registers manifest once
Artifact ID != Asset ID retained in provenance
no provider-specific runtime action injection
existing AssetCompiler/Admission tests remain green
```

## 14. Merge Gate

- no changes to Physics/Interaction/WorldRuntime required;
- reuse existing AssetCompiler/Admission APIs;
- A-01/A-02 regressions green;
- full JS regression PASS;
- production build PASS;
- Python smoke PASS;
- CodeGraph confirms bounded pipeline impact;
- GitHub Actions build + Pages deploy PASS.

## 15. Completion Evidence

```text
Final Status:       MERGED
Final Commit:       162a101 feat: add verified artifact asset production loop
Merged into main:   yes
Focused tests:      6 files / 51 tests PASS
Wide regression:    13 files / 88 tests PASS
Full JS regression: 133 files / 551 tests PASS
Asset validation:   PASS
Production build:   PASS (exit_code=0, 15.84s)
Python smoke:       4/4 PASS
CodeGraph impact:   5 nodes / 6 edges
GitHub Actions run: 32708197749
Test and build:     success
Pages deploy:       success
```

Key correctness boundaries implemented:

- verified Artifact local-cache identity is checked before compiler invocation;
- Connector bytes are not re-fetched once verified local bytes exist;
- Artifact ID and Asset ID remain distinct;
- compile lease is held across compile/admission/register and always released;
- real generic GLB compiler fallback remains `asset-provisional`;
- compiler rejection and admission rejection do not fabricate registration success;
- compiler failure does not mutate Artifact integrity or evict verified bytes;
- same source Artifact/hash may safely reuse an already-registered asset; conflicting assetId provenance fails closed;
- compiler-generated actions/physics remain authoritative; provider metadata does not inject runtime actions.

### Next actual gap

Current repository inspection shows AS-06/07 part-segmentation bridge is already implemented, but `source_urdf` remains descriptor-only in `EmbodiedGenBundleAdapter`; URDF bytes are not yet hash-verified/parsing-normalized into compiler evidence. The next slice should close this gap rather than duplicate the existing bundle/segmentation path.
