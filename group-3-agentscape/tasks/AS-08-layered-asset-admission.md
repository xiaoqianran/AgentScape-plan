# AS-08 — Layered Asset Admission Contract

## 1. Task Identity

```text
Task ID:        AS-08
Parent Plan:    AgentScape Integration Plan / Asset Admission 收敛
Status:         MERGED
Base Commit:    571c747
Final Commit:   5202f6f
Feature Branch: feat/as08-layered-admission
```

## 2. Goal

Replace the current single-source/override admission decision with a deterministic layered decision across provider, compiler and runtime verification evidence.

Core rule:

```text
provider status
compiler status
runtime status
      |
      v
aggregate = worst(required layers)
```

Ordering:

```text
rejected > provisional > ready
```

Any required layer rejected => aggregate rejected.
Any required layer provisional/unknown => aggregate provisional.
Only all required layers ready => aggregate ready.

## 3. Current Bug

Current `assetAdmission()` returns `provenance.admission` immediately when present. Therefore an explicit provider `ready` may hide a compiler `rejected`, and a provider `provisional` may hide a stronger compiler/runtime rejection.

This violates AS-08.

## 4. Layer Sources

### Provider layer

Use explicit `manifest.provenance.admission` as provider-layer evidence only, not aggregate authority.

When provider evidence exists without explicit admission, derive safe provider provisional reasons from provider evidence levels where necessary.

### Compiler layer

Use `manifest.compiler.quality`:

```text
ready       -> ready
provisional -> provisional + advisory codes
rejected    -> rejected + hard codes or COMPILER_REJECTED
missing     -> for generated assets: provisional / COMPILER_UNVERIFIED
```

### Runtime layer

Runtime layer is required only when executable runtime capability needs verification, initially articulation.

Derive from:

```text
manifest.verification.articulation.ok === true  -> ready
manifest.verification.articulation.ok === false -> provisional/rejected according to policy
ARTICULATION_UNVERIFIED advisory                  -> provisional
ARTICULATION_VERIFICATION_FAILED advisory         -> provisional
```

Do not infer runtime-ready merely from provider/SAPIEN evidence.

## 5. Output Contract

Keep backwards-compatible top-level shape:

```text
{
  status: ready | provisional | rejected,
  reasons: [...deduped codes]
}
```

Add optional/always-present diagnostics:

```text
layers: {
  provider: { status, reasons },
  compiler: { status, reasons },
  runtime:  { status, reasons, required }
}
```

Callers that only read `.status/.reasons` continue to work.

## 6. Reason Taxonomy

Prefer existing stable codes; do not churn names unnecessarily.

Provider examples:
- `FALLBACK_BOX_COLLIDER`
- `UNVERIFIED_PROVIDER_SEMANTICS`
- `PROVIDER_GRASP_UNVERIFIED`
- `PROVIDER_GRASP_RAW_ONLY`
- `PROVIDER_GRASP_SAPIEN_ONLY`

Compiler examples:
- hard compiler issue codes
- compiler advisory codes
- `COMPILER_PROVISIONAL`
- `COMPILER_REJECTED`
- `COMPILER_UNVERIFIED`

Runtime examples:
- `ARTICULATION_UNVERIFIED`
- `ARTICULATION_VERIFICATION_FAILED`

## 7. Required Safety Semantics

- explicit provider ready cannot override compiler rejected;
- explicit provider provisional cannot hide compiler rejected;
- compiler ready cannot override provider rejected;
- provider/SAPIEN grasp evidence never makes runtime layer ready;
- successful articulation verification removes only articulation-runtime provisional reason;
- failed articulation verification never becomes ready;
- non-generated builtin/repo assets without layered metadata preserve current ready behavior;
- generated manifests without compiler evidence remain provisional.

## 8. Call-Site Compatibility

Audit at least:

```text
AssetLibrary.generate
createWorldPipeline
VerifiedArtifactAssetPipeline
spawnAsset skill
importEmbodiedGenAsset skill
verifyAssetArticulation skill
```

No caller should rely on explicit provider admission being aggregate authority.

## 9. Required Tests

At minimum:

```text
provider ready + compiler rejected -> rejected
provider provisional + compiler rejected -> rejected
provider rejected + compiler ready -> rejected
provider provisional + compiler ready -> provisional
provider ready + compiler provisional -> provisional
all required layers ready -> ready
generated missing compiler -> provisional
builtin no layered metadata -> ready
runtime articulation unverified -> provisional
runtime articulation verified -> ready when last blocker
runtime articulation failed -> provisional
layer reasons deduped/stable
existing AS-05 production pipeline behavior preserved
existing spawn/generate/admission tests remain green
```

## 10. Non-goals

- no new Runtime verifier;
- no new grasp execution verifier;
- no World mutation changes;
- no provider-specific admission bypass;
- no change to Compiler quality generation itself unless required for reason plumbing.

## 11. Merge Gate

- focused admission/runtime tests PASS;
- AS-05/AS-06 regressions PASS;
- full JS regression PASS;
- build PASS;
- Python smoke PASS;
- CodeGraph impact reviewed;
- GitHub Actions build + Pages deploy PASS.

## 12. Completion Evidence

```text
Final Status:       MERGED
Final Commit:       5202f6f feat: converge layered asset admission
Merged into main:   yes
Focused tests:      6 files / 60 tests PASS
Wide regression:    13 files / 100 tests PASS
Full JS regression: 133 files / 576 tests PASS
Asset validation:   PASS
Production build:   PASS (vite exit_code=0, 16.47s)
Python smoke:       7/7 PASS
CodeGraph impact:   56 nodes / 92 edges
GitHub Actions run: 32714571014
Test and build:     success
Pages deploy:       success
```

Key convergence rules now enforced:

- provider/compiler/runtime statuses are separate layers;
- aggregate readiness uses worst-layer-wins;
- explicit provider admission is no longer aggregate authority;
- legacy AS-05 aggregate snapshots are detected and ignored as provider evidence;
- generated assets without compiler evidence are `COMPILER_UNVERIFIED`;
- provider/SAPIEN grasp evidence cannot satisfy runtime articulation readiness;
- runtime articulation success removes only the runtime blocker;
- runtime articulation failure remains provisional;
- `verifyAssetArticulation` now returns aggregate admission readiness rather than raw compiler quality;
- AS-05 no longer writes the aggregate admission back to top-level `provenance.admission`.
