# AgentScape × EmbodiedGen Evidence Bridge 执行计划

> 更新时间：2026-08-24
>
> 本文只定义 EmbodiedGen enriched bundle 如何进入 AgentScape 现有 Compiler / Admission。核心原则：**provider evidence 进入编译器，不直接变成 runtime truth。**

> **2026-08-24 实施状态**：core bridge 已验证。Provider 基线 `modal-build@adf9fcf`；AgentScape 实现 `671e1ac feat: bridge EmbodiedGen evidence into compiler`。真实 50k-face Bundle 已得到 `materialized` / coverage=1 / 4 parts，并正确保持 `provisional`。

## 1. 当前 AgentScape 事实

当前代码已经具备：

- `src/compiler/AssetCompiler.js`
  - `compile({ url, bytes, sourceName, assetId, label, partProposal, partSegmentation, providerEvidence })`
- `src/compiler/passes/SegmentationEvidencePass.js`
- `src/compiler/passes/SegmentMaterializePass.js`
- `src/compiler/passes/PartProposalPass.js`
- `src/compiler/passes/CompileQualityPass.js`
- `src/assets/admission.js`

当前 `src/adapters/EmbodiedGenAdapter.js` 仍是 legacy adapter：

- 输入 shape 宽松；
- 要求 browser-reachable GLB URL；
- 直接生成 runtime Manifest；
- collider fallback 为 box；
- provider affordance 字符串可能被提升为 `pickup/drop/place`；
- admission 始终 provisional。

这个 legacy path 可继续兼容旧调用，但 **不应该承接新的 Affordance bundle**。

## 2. 新主路径

```text
EmbodiedGen Job/Bundle
      |
      | authenticated descriptors + bytes + hashes
      v
EmbodiedGenBundleAdapter
      |
      +--> visual GLB bytes --------------------+
      |                                         |
      +--> part segmentation evidence ----------+--> AssetCompiler
      |                                         |
      +--> semantic part proposal (optional) ---+
      |                                         |
      +--> provider evidence summary ------------+
                                                |
                                                v
                                      Compiler Quality
                                                |
                                                v
                                         Asset Admission
                                                |
                                 +--------------+--------------+
                                 |              |              |
                               ready        provisional      rejected
```

Adapter 不返回最终“truth Manifest”；它返回 Compiler input + provider evidence。

## 3. P3-SAM → `partSegmentation` 精确映射

AgentScape `SegmentationEvidencePass` 当前要求：

```text
version = 1
source
faceCount
segments[]:
  id
  faceCount
  optional confidence/semantic/bounds
optional artifact
optional materialization
```

`SegmentMaterializePass` 如果执行 provider face partition，还要求：

```text
materialization.sourceNode
materialization.primitives[]:
  primitive
  faceLabels[]
```

### 3.1 Provider alignment Gate 已实现并验证

`modal-build@adf9fcf` 已实现并通过 derived-job canary 的该硬 Gate：P3-SAM flat source labels 不直接交给 AgentScape，而是在 provider 端重新解析 primary GLB，先验证 OBJ/GLB vertex identity，再按 triangle vertex-index set 建立 face mapping，最终发布 `agentscape_part_segmentation.v1.json`。

真实 production evidence：

- 50,000 source faces → 50,000 final GLB triangle labels；
- sourceNode=`geometry_0`；
- one TRIANGLES primitive；
- primary GLB SHA=`4990691a19e7abcfd7c67853fb907b55792c133635631105eccdda6f2aae1861`；
- vertex max abs error ≈ `5e-9`；
- duplicate/missing/out-of-range mapping 全部 fail closed。

因此 Adapter 继续只做 schema/hash/identity 验证，不负责猜测或重建 face mapping。

### 3.2 目标转换

当 provider 输出 `agentscape_part_segmentation.v1.json` 后，Adapter 做机械转换：

```text
provider.version            -> partSegmentation.version
provider.source             -> partSegmentation.source
provider.faceCount          -> partSegmentation.faceCount
provider.segments           -> partSegmentation.segments
provider.glb.sha256         -> partSegmentation.artifact.sha256
provider.sourceNode         -> partSegmentation.materialization.sourceNode
provider.primitives         -> partSegmentation.materialization.primitives
```

Adapter 只做 schema/hash/identity 验证，不修改 labels。

## 4. Semantic → `partProposal` 映射

只有 GPT semantic artifact 真实存在时才生成 `partProposal`。

允许映射：

- provider part ID；
- materialized node identity；
- semantic name/category/description；
- semantic confidence/provenance；
- parent relation（仅当 provider 有明确结构证据）。

禁止从语义猜测：

- revolute / prismatic joint type；
- axis；
- anchors；
- limits；
- open/close target；
- collider；
- `pickup` 已成功。

因此 semantic-only `partProposal` 预期会被 `PartProposalPass` 接受为 evidence，但因为缺少 executable joint/physics 条件，大部分 part 应保持 `unpromoted`。这是正确结果。

## 5. Grasp evidence 的位置

raw GraspGen 与 SAPIEN validated grasp 都不直接对应当前 `PartProposalPass` 的 executable articulation contract。

当前 `671e1ac` 先保存 **小型 evidence summary + artifact descriptor**，而不把 raw grasp pose 数组内嵌进 Manifest：

```text
providerEvidence.levels.grasps = raw-provider-only | sapien-validated-provider-only | none
providerEvidence.artifacts[] = { role, sha256, bytes, mediaType, verified, ... }
```

真实 raw grasp payload 继续由 provider artifact 按 hash 引用。当前真实 provenance summary 约 906 bytes，没有 `faceLabels[]` 或 raw grasp pose 大数组。它们可供 UI、后续 interaction planner 或专用 verifier 使用，但不能因为 `sapien-validated` 就直接改变 AgentScape Runtime action truth。

## 6. 具体任务

### AS-EG-01：新增 `EmbodiedGenBundleAdapter`

**状态：VERIFIED — `AgentScape@671e1ac`。**

**仓库**：`AgentScape`

**建议文件**：

- 新增 `src/adapters/EmbodiedGenBundleAdapter.js`
- 新增 `tests/embodiedgen-bundle-adapter.test.js`

职责：

1. 校验 bundle version；
2. 校验 required artifact descriptor；
3. 校验 SHA-256 / MIME / role；
4. 提取 visual GLB bytes/descriptor；
5. 提取 segmentation / semantic / grasp evidence；
6. 返回 Compiler input；
7. 不直接生成最终 Manifest。

保留 `EmbodiedGenAdapter.js`，明确标记为 legacy/provisional compatibility path。

**验收**：坏 hash、未知 schema、缺 primary GLB、segmentation 指向错误 GLB SHA 时全部 fail closed。

### AS-EG-02：P3-SAM Evidence Transformer

**状态：VERIFIED core — 真实 50k-face provider artifact 已通过 BundleAdapter→AssetCompiler。**

**建议文件**：

- `src/adapters/EmbodiedGenBundleAdapter.js`
- `tests/embodiedgen-bundle-adapter.test.js`
- `tests/asset-compiler-segmentation-provider.test.js`

职责：把 provider 的 compiler-native segmentation artifact 转成现有 `partSegmentation` shape。

必须验证：

- version/source；
- GLB SHA identity；
- faceCount；
- segments 唯一；
- primitive index 唯一且覆盖；
- 每个 labels 数量由 Compiler 再次验证。

**E2E 验收**：冻结一个真实 P3-SAM provider fixture，送入 AssetCompiler 后：

- `SegmentationEvidencePass.issues=[]`；
- `SegmentMaterializePass.materialization.status='materialized'`；
- materialized part count 与 provider part count 一致；
- compile quality 不因 segmentation contract 错误 rejected。

### AS-EG-03：Provider Evidence Provenance

**状态：VERIFIED — providerEvidence summary 已进入 Manifest provenance，且 Secret/signed URL/path traversal fail closed。**

`671e1ac` 已给 `AssetCompiler.compile()` 增加 `providerEvidence` 参数，并将小型 summary 安全带入 `ManifestPass` provenance。

建议：

- `compile({ ..., providerEvidence = null })`；
- Context 只携带结构化、已大小限制的 summary；
- `ManifestPass` 保存 lineage/evidence summary，不内嵌巨大的 `faceLabels` / raw grasp arrays；
- 大 evidence 用 artifact ID/hash 引用。

**建议文件**：

- `src/compiler/AssetCompiler.js`
- `src/compiler/passes/ManifestPass.js`
- 对应 compiler tests

安全要求：signed URL、Authorization、Secret、原始 GPT credential 永远不进入 Manifest。

### AS-EG-04：Admission reason 分层

**状态：VERIFIED — versioned provider bundle 可得到具体 provisional reasons；legacy admission 行为保持兼容。**

**建议文件**：

- `src/assets/admission.js`
- `tests/asset-admission.test.js`

需要能表达至少：

- `PROVIDER_PART_SEGMENTATION_VERIFIED`
- `PART_SEMANTICS_UNVERIFIED`
- `PROVIDER_GRASP_RAW_ONLY`
- `PROVIDER_GRASP_SAPIEN_ONLY`
- `ARTICULATION_UNVERIFIED`
- `COMPILER_PART_MATERIALIZATION_FAILED`

注意：positive evidence 不必全部变成 admission reason；reason 主要解释为什么 provisional/rejected。名称最终以现有 reason convention 收敛。

### AS-EG-05：真实 frozen fixture E2E

**状态：PLANNED。真实 production E2E 已手工验证，但脱敏 frozen fixture 尚未提交进测试仓库。**

将一个脱敏、体积受控的真实 provider 输出冻结进测试 fixture：

- primary/segmented GLB 或最小等价 fixture；
- provider bundle descriptor；
- compiler-native segmentation JSON；
- expected part count / face counts / hash。

测试路径：

```text
bundle fixture
-> EmbodiedGenBundleAdapter
-> AssetCompiler
-> SegmentationEvidencePass
-> SegmentMaterializePass
-> CompileQualityPass
-> assetAdmission
```

首个正确目标允许是：

`provisional`

原因可以是 collider/articulation/semantics 未验证。不要为了让测试得到 `ready` 而绕开门禁。

### AS-EG-06：Semantic / Grasp 增量接入

**状态：PARTIAL。`raw_grasps` role 已可作为 `raw-provider-only` descriptor evidence；payload validation、semantic 与 SAPIEN 层仍待实现。**

等 provider 分别发布：

- `part_semantics.v1`
- `raw_grasps.*.v1`
- `sapien_grasps.*.v1`

再逐层扩 Adapter。每增加一层必须有独立 fixture 和 admission expectation。

## 7. 文件级任务顺序

```text
AgentScape:

1. src/adapters/EmbodiedGenBundleAdapter.js
2. tests/embodiedgen-bundle-adapter.test.js
3. tests/asset-compiler-segmentation-provider.test.js
4. src/compiler/AssetCompiler.js              (仅在 providerEvidence 确有需要后)
5. src/compiler/passes/ManifestPass.js
6. src/assets/admission.js
7. tests/asset-admission.test.js
8. AssetLibrary / Skill / UI orchestration      (最后)
```

不要先修改 Skill/UI，因为 bundle/Compiler contract 尚未冻结。

## 8. 跨仓 Gate

### Provider Gate

`modal-build` 必须先提供：

- immutable artifact identities；
- compiler-native GLB primitive face labels；
- stable provider schema version；
- hash/lineage；
- validation report。

### Compiler Gate

AgentScape 必须证明：

- segmentation materialization 精确；
- provider semantic 不越权变成 articulation truth；
- raw/SAPIEN grasp 不越权变成 interaction truth；
- provider success 仍可得到 compiler `provisional/rejected`。

### Runtime Gate

只有 Compiler/Admission 已 ready 的执行语义，才允许 Agent/Runtime 对用户声称可执行。Provider evidence 可以被展示和用于 planner proposal，但必须保留 evidence level。

## 9. 与当前 legacy adapter 的迁移关系

旧 `EmbodiedGenAdapter.toManifest(payload,{glbUrl})`：

- 保留；
- 标记 compatibility；
- 继续 provisional；
- 不增加新的 Affordance 逻辑；
- 不再把新 bundle 特性塞进它的 loose payload 分支。

新 `EmbodiedGenBundleAdapter`：

- 面向 versioned bundle；
- bytes/hash-first；
- Compiler-first；
- evidence-preserving；
- 是后续 AgentScape 与 EmbodiedGen 的正式主路径。

## 10. 2026-08-24 验证证据

跨仓真实 E2E（非 synthetic fixture）：

- provider：`modal-build@adf9fcf`；
- consumer：`AgentScape@671e1ac`；
- source Job：`job-f82e3eaab6a846e08d32874788495b80`；
- derived Affordance Job：`job-a2595a4645f6454cb9d4dbc2b0dff692`，profile=`part-evidence-only`；
- derived stage seconds：segment=37.561 / grasp_raw=19.163 / finalize=3.899；
- bundle v1 的 primary GLB / URDF / segmentation / raw grasp descriptor SHA 均独立复算匹配；
- BundleAdapter 校验 primary GLB + segmentation SHA，raw grasp 保留 descriptor-level evidence；
- `SegmentMaterializePass.status=materialized`；
- coverage=`1`；
- materialized parts=`4`；
- Compiler hard findings=`0`；
- final quality=`provisional`；
- admission reasons 包含 `PART_SEMANTICS_UNVERIFIED`、`PROVIDER_GRASP_RAW_ONLY`；
- provenance providerEvidence summary≈906 bytes，无大数组；
- AgentScape split test suite：shard 1/2 = 57 files / 190 tests PASS；shard 2/2 = 57 files / 216 tests PASS；
- `npm run assets:validate` PASS；
- `npm run build` PASS。

因此 **Core Evidence Bridge 已验证**；完整阶段收口仍等待 AS-EG-05 frozen fixture、semantic/SAPIEN 增量与正式 Artifact/Job transport。

## 11. 完成定义

第一阶段 Evidence Bridge 完成，不要求完整 GPT/SAPIEN Affordance。必须同时满足：

1. provider P3-SAM artifact 已严格对齐 final GLB primitives；
2. AgentScape bundle adapter fail closed；
3. 真实 fixture 可被 `SegmentMaterializePass` materialize；
4. provider part count 与编译后 materialized part count 一致；
5. compile/admission 状态可解释；
6. legacy adapter 未被破坏；
7. provider semantics/grasp 不会自动生成未经验证的 joint/action；
8. tests + build 全绿。
