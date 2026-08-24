# AS-06B — Grasp Evidence Integrity Gate

## 1. Task Identity

```text
Task ID:        AS-06B
Parent Plan:    AS-06 / AS-08 evidence boundary
Status:         MERGED
Base Commit:    a4b3297
Final Commit:   571c747
Feature Branch: feat/as06b-grasp-evidence-integrity
```

## 2. Goal

Turn provider grasp artifacts from descriptor-only hints into hash/schema-verified evidence without promoting them into AgentScape pickup truth.

## 3. Completed Contract

- raw/SAPIEN grasp JSON is optional evidence;
- descriptor without bytes remains `*-provider-unverified`;
- bytes are bounded to 2 MiB and SHA-256 verified;
- JSON schema validates version/evidence level/job lineage/gripper/source frame/rank/score;
- grasp pose must be finite rigid 4x4 affine transform with right-handed orthonormal rotation;
- candidate count <= 256 and ranks unique;
- `source_job_id` must match bundle `sourceJobId` when present;
- raw verified evidence maps to `raw-provider-only`;
- SAPIEN verified evidence maps to `sapien-validated-provider-only`;
- verified raw remains authoritative if higher-level SAPIEN descriptor has no bytes;
- provider evidence provenance stores only summary metadata, not pose arrays;
- pickup/held/runtime-verified/actions/secrets/url/path/base64-style fields are rejected;
- provider grasp evidence never creates pickup actions or Runtime held state.

## 4. Completion Evidence

```text
Focused JS:          3 files / 33 tests PASS
Wide JS:             10 files / 79 tests PASS
Full JS regression:  133 files / 568 tests PASS
Asset validation:    PASS
Production build:    PASS (vite exit_code=0, 13.73s)
Python endpoint/unit: 7/7 PASS
GitHub Actions run:  32713433091
Test and build:      success
Pages deploy:        success
```

## 5. Next Actual Gap

Evidence integrity is now materially stronger, but `assetAdmission()` still allows a single explicit `provenance.admission` object to override compiler/runtime truth. This violates AS-08's rule that any rejected layer must reject the aggregate asset. Next task: AS-08 Layered Admission Contract.
