# AgentScape × EmbodiedGen Evidence Bridge 执行计划

> 更新时间：2026-08-24
>
> 本文只定义 EmbodiedGen enriched bundle 如何进入 AgentScape 现有 Compiler / Admission。核心原则：**provider evidence 进入编译器，不直接变成 runtime truth。**

## 1. 当前 AgentScape 事实

当前代码已经具备：

- `src/compiler/AssetCompiler.js`
  - `compile({ url, bytes, sourceName, assetId, label, partProposal, partSegmentation })`
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

### 3.1 当前 provider 输出不能直接无条件映射

当前 P3-SAM runtime 已有：

- `face_count`
- `part_count`
- `part_face_counts`
- flat `face_ids[]`
- `mesh_part_seg.glb`

但 flat `face_ids[]` 是针对 provider source OBJ face order。Compiler 输入是最终 GLB，其 node / primitive 划分和 triangle order 必须单独证明一致。

因此硬 Gate：

> provider 必须发布 **与最终 Compiler GLB primitive 顺序对齐** 的 faceLabels artifact；Adapter 不负责猜测或重建对应关系。

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

首版应保存在 provider provenance/evidence，例如：

```text
providerEvidence.affordance.grasps[]
  partId
  gripper
  pose
  score
  level = raw | semantic-selected | sapien-validated
  sourceFrame
  modelRevision
  simulationProfile (if any)
```

它们可供 UI、后续 interaction planner 或专用 verifier 使用，但不能因为 `sapien-validated` 就直接改变 AgentScape Runtime action truth。

## 6. 具体任务

### AS-EG-01：新增 `EmbodiedGenBundleAdapter`

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

当前 `AssetCompiler.compile()` 没有单独的 `providerEvidence` 参数。新增前先确定最小存储边界。

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

## 10. 完成定义

第一阶段 Evidence Bridge 完成，不要求完整 GPT/SAPIEN Affordance。必须同时满足：

1. provider P3-SAM artifact 已严格对齐 final GLB primitives；
2. AgentScape bundle adapter fail closed；
3. 真实 fixture 可被 `SegmentMaterializePass` materialize；
4. provider part count 与编译后 materialized part count 一致；
5. compile/admission 状态可解释；
6. legacy adapter 未被破坏；
7. provider semantics/grasp 不会自动生成未经验证的 joint/action；
8. tests + build 全绿。
