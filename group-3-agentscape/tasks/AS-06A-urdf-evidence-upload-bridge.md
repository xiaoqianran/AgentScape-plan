# AS-06A — Verified URDF Evidence Upload Bridge

## 1. Task Identity

```text
Task ID:        AS-06A
Parent Plan:    AS-06 / AS-07 evidence bridge
Status:         MERGED
Base Commit:    162a101
Final Commit:   a4b3297
Feature Branch: feat/as06a-urdf-evidence-upload
```

## 2. Goal

Close the existing `source_urdf` gap without inventing a second compiler or browser XML parser:

```text
verified source_urdf bytes
  -> Asset Compiler multipart stage=urdf-proposal
  -> yourdfpy Part Proposal v1
  -> browser-side trust/schema gate
  -> existing SegmentMaterialize / JointFrame / PartProposal / Quality passes
```

## 3. Completed Boundary

- `/compile` accepts bounded multipart `stage=urdf-proposal` uploads;
- upload is parsed by existing `urdf_part_proposal()` / `yourdfpy`;
- multipart UploadFile is always closed in `finally`;
- `HttpCompilerProvider.runUrdfProposal(bytes)` reuses the existing compiler endpoint;
- `EmbodiedGenBundleAdapter` verifies URDF bytes + SHA-256 before parsing;
- browser-side URDF bytes have a 5 MiB hard cap;
- unconfigured parser leaves evidence at `verified-bytes-only`;
- valid parser response becomes `service-parsed` Part Proposal candidate;
- invalid/malicious parser response becomes `parse-rejected` and does not enter Compiler;
- proposal is limited to mechanical evidence only;
- `actions`, `physics`, `targets`, motor/secret/url/path/prompt/remote-id style fields are rejected;
- part IDs, child nodes, parent references and hierarchy cycles are validated;
- revolute/prismatic/continuous joint type, finite axis/limits and 4x4 matrices are validated;
- existing `JointFramePass` remains the authority for whether URDF geometry can produce executable anchors;
- real cabinet scale mismatch remains `JOINT_FRAME_SCALE_UNSUPPORTED` / provisional instead of inventing anchors.

## 4. Completion Evidence

```text
Focused JS:          4 files / 25 tests PASS
Wide JS:             10 files / 63 tests PASS
Full JS regression:  133 files / 560 tests PASS
Asset validation:    PASS
Production build:    PASS (vite exit_code=0, 14.15s)
Python endpoint/unit: 7/7 PASS
CodeGraph:
  BundleAdapter      5 nodes / 8 edges
  runUrdfProposal    5 nodes / 4 edges
GitHub Actions run:  32710437634
Test and build:      success
Pages deploy:        success
```

## 5. Remaining AS-06/07 Gap

URDF is now a mechanically validated evidence source. Remaining evidence maturity should focus on evidence that is still descriptor-only or explicitly raw, especially grasp evidence, and on admission/runtime verification rules that decide whether any inferred articulation/grasp may become executable.
