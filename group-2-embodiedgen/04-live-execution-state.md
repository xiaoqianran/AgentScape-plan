# EmbodiedGen × `modal-build` 实时执行状态与下一任务

> 更新时间：2026-08-24
>
> 这份文档是 **执行态事实源**，不替代 `01-modal-build-plan.md` 的长期架构。它只回答四个问题：现在真实完成了什么、当前正在做什么、下一步具体改哪些文件、什么证据才允许把任务标成完成。

## 1. 状态语义

本文只使用以下四种状态：

- **VERIFIED**：存在真实 Action / Modal / 测试 / artifact 证据，且证据与声明直接对应。
- **IMPLEMENTED / UNVERIFIED**：代码或文档已经存在，但缺少目标环境真实验证，不能对外声称完成。
- **GATED**：下一步依赖外部前置条件，例如 Secret、模型许可、运行预算或上游接口。
- **PLANNED**：只有任务定义，还没有实现证据。

禁止把“代码写了”“测试字符串存在”“Modal function 返回 200”单独等价成 VERIFIED。

## 2. 当前事实快照

### 2.1 `modal-build` 基线

远端 `master` 当前正式基线已经推进到：

- `69c910c add validated EmbodiedGen affordance core runtime`
- 该提交在 `ef6e4cc add EmbodiedGen affordance CUDA artifact builder` 之上完成 P3-SAM runtime、GraspGen raw inference、模型权重供应链与 AgentScape-native segmentation evidence；
- 更早生产验证：
  - Image→3D：`79370bf document warning-free EmbodiedGen production E2E`
  - Text→3D：`2ca3355 document production Text-to-3D validation`
  - Retexture：`96a8d65 document production EmbodiedGen retexture validation`

`69c910c` 提交前本地测试为 **74/74 PASS**，并已推送到 `origin/master`。因此以下 Affordance core 能力已经不是 dirty experiment，而是有远端提交与真实 Modal/L40S 证据的正式基线。

### 2.2 已验证的 Affordance 供应链

原生 CUDA 依赖已经在真实 NVIDIA L40S / SM89 / Torch 2.8.0+cu126 上执行 kernel 验证：

- `torch-scatter 2.1.2`
- `pointnet2_ops 3.0.0`
- `chamfer_3D 0.0.0`

固定 release：

`embodiedgen-v2.0.0-affordance-py310-cu126-torch280-sm89-v1`

当前已验证原则：

- build worker 不要求 GPU；
- consumer runtime 不携带 `nvcc`；
- wheel 使用 SHA-256 校验；
- EmbodiedGen / GraspGen / Hunyuan3D-Part 均固定 commit；
- EmbodiedGen gitlink 与独立 dependency pin 必须一致；
- P3-SAM / Sonata 禁用 Flash Attention；
- runtime 不在启动时动态编译 CUDA extension。

### 2.3 P3-SAM part segmentation 已真实 E2E

验证输入：

`job-f82e3eaab6a846e08d32874788495b80`

真实 Modal L40S 结果：

- GPU：NVIDIA L40S
- Compute Capability：8.9
- Torch：2.8.0+cu126
- CUDA：12.6
- source mesh：50,000 faces
- segmentation labels：50,000 faces
- part count：4
- model load：1.629 s
- inference：18.926 s
- result marker：`P3SAM_PART_SEGMENTATION_OK`

已写入 job artifact：

- `affordance/mesh_part_seg.glb`
- `affordance/part_segmentation.json`
- `affordance/validation_report.json`
- `affordance/agentscape_part_segmentation.v1.json`

compiler-native evidence 已额外验证：

- primary GLB SHA-256：`4990691a19e7abcfd7c67853fb907b55792c133635631105eccdda6f2aae1861`；
- sourceNode：`geometry_0`；
- primitive count：1；
- 50,000 GLB triangles 全部对应到 source OBJ face；
- vertex identity 最大绝对误差约 `5e-9`；
- mapping strategy：`verified-vertex-identity-triangle-index-set`；
- 任一 vertex drift、duplicate triangle、missing face 都 fail closed。

因此 AFF-02 的核心 Gate 已通过：**P3-SAM labels 已与最终 primary GLB primitive triangle order 明确绑定**，不再依赖 face count 或颜色猜测。

它仍然不证明：

- part 有真实 joint；
- semantic label 正确；
- raw grasp 等于可执行 pickup；
- AgentScape/Rapier 已验证这些 grasp。

### 2.4 GraspGen raw 6-DoF inference 已真实 E2E

固定 Hugging Face revision：

`ec1ccbb5eec0680db669246ac312a3636f16ee43`

Franka Panda 文件：

| 文件 | bytes | SHA-256 | 当前状态 |
|---|---:|---|---|
| `graspgen_franka_panda.yml` | 4,868 | `3b666d28ffb91001ddb6ba24a2e0c11458478a986b808b493cf6fa9a987c2abd` | VERIFIED download |
| `graspgen_franka_panda_gen.pth` | 907,408,223 | `0597583b89b322d42ceb4e596967d6ed68d1b56cba4039895909ccd5bdc66eff` | VERIFIED download |
| `graspgen_franka_panda_dis.pth` | 165,853,892 | `e47d703c63b54c2d11fbc1effd43898f251b4147250888541e3b16e9c0d19e1c` | VERIFIED download |

GraspGen 上游 generator / discriminator 已显式 `enable_flash=False`。`69c910c` 已新增 raw inference worker，并对同一个 production Job 做真实 L40S 验证：

- 输入来自 production URDF collision mesh，并解析 mesh `scale`、collision `origin xyz/rpy`；
- generator + discriminator 均从固定 revision/固定 SHA 权重离线加载；
- 输出 `affordance/raw_grasps.franka.v1.json`；
- top-20 raw grasps；
- confidence 范围约 `0.50379 .. 0.86179`；
- rotation orthogonality max error 约 `2.38e-7`；
- rotation determinant max error 约 `4.77e-7`；
- model load 约 `2.552 s`；
- inference 约 `0.868 s`；
- result marker：`GRASPGEN_RAW_GRASPS_OK`。

这证明的是 **GraspGen raw candidate generation 可运行**。artifact 明确标记 `evidence_level=raw`，不代表 semantic-selected、SAPIEN-validated 或 AgentScape pickup verified。

### 2.5 GPT semantic stage 当前 GATED

EmbodiedGen 原生完整 Affordance 链：

```text
part segmentation
  -> part semantic annotation
  -> grasp generation/filtering
  -> SAPIEN grasp evaluation
  -> enriched URDF/bundle
```

当前 Modal 只存在 `github` 与 `huggingface` Secret，没有配置语义 worker 所需 GPT endpoint Secret。

因此：

- P3-SAM core 可以继续产品化；
- GraspGen raw inference 已验证，可作为后续 semantic selection / SAPIEN evaluation 的输入；
- `partsemantics_annot.py` 真实 E2E 暂时是 GATED；
- 不允许伪造 `affordance.json` 来让完整 pipeline “变绿”。

## 3. 当前能力矩阵

| 能力 | 当前状态 | 已有证据 | 下一个 Gate |
|---|---|---|---|
| Image→3D | VERIFIED production | production E2E commit | 迁移到统一 workflow contract |
| Text→3D | VERIFIED production | production validation commit | 迁移到统一 workflow contract |
| Retexture | VERIFIED production | production validation commit | artifact/contract 收敛 |
| Affordance native wheels | VERIFIED | L40S kernel smoke + immutable release | 已由 `69c910c` runtime 消费 |
| P3-SAM segmentation | VERIFIED | 50k faces / 4 parts / L40S report | semantic stage |
| GraspGen weights | VERIFIED | exact revision + bytes + SHA + runtime load | semantic/eval profiles |
| GraspGen raw inference | VERIFIED | production URDF transform + 20 raw grasps + finite pose checks | semantic selection / SAPIEN eval |
| GPT part semantics | GATED | 无 GPT Secret | 配置独立 semantic Secret / endpoint |
| SAPIEN grasp evaluation | PLANNED | 无 runtime canary | raw grasps + semantic stage |
| Affordance Job API | PLANNED | 目前是独立 worker | stage contract / job state / artifact roles |
| AgentScape part evidence import | VERIFIED core | `671e1ac` BundleAdapter + real 50k-face compile/admission E2E | frozen fixture + transport integration |

## 4. 执行任务图

```text
                     已验证 production 3D Job
                              |
                   OBJ + URDF + source metadata
                              |
                              v
                 [AFF-01 P3-SAM segmentation] -------- VERIFIED
                              |
                +-------------+----------------+
                |                              |
                v                              v
 [AFF-02 compiler-native labels] ✅     [AFF-03 raw GraspGen] ✅
  final GLB primitive 对齐 VERIFIED      URDF transform + 6DoF VERIFIED
                |                              |
                v                              v
      [AS-EG-02 Evidence Bridge] ✅     raw grasp evidence
                |                              |
                +---------------+--------------+
                                |
                    [AFF-04 semantic worker]
                       GPT Secret GATE
                                |
                                v
                    semantic part proposal
                                |
                                v
                    [AFF-05 SAPIEN eval]
                                |
                                v
                  sim-filtered grasp evidence
                                |
                                v
                  [AFF-06 immutable bundle]
                                |
                                v
                 [AS-EG-03 Compile/Admission] ✅ core
                                |
                                v
                   AgentScape provisional/ready
```

关键原则：AgentScape segmentation bridge 与 raw GraspGen 可以并行；不需要等待 GPT semantic 才开始消费 part evidence。

## 5. 下一批具体任务

### AFF-02：生成 Compiler-native segmentation evidence

> **状态：VERIFIED（2026-08-24）**。实现已进入 `modal-build@69c910c`，真实 production Job 已生成 `agentscape_part_segmentation.v1.json`，并被 AgentScape Compiler materialize 成 4 个 parts、coverage=1。

**目的**：解决当前最重要的数据契约风险——P3-SAM face IDs 是针对 provider source mesh 产生的，AgentScape `SegmentMaterializePass` 要求最终 GLB 每个 primitive 的 `faceLabels` 与 triangle 顺序严格一致。

**仓库**：`modal-build`

**主要文件**：

- `runtime/embodiedgen_affordance_l40s.py`
- `tests/test_embodiedgen_affordance_runtime.py`

**目标输出**：新增 provider artifact，例如：

`affordance/agentscape_part_segmentation.v1.json`

最低内容：

- `version=1`
- `source="embodiedgen/p3sam"`
- `faceCount`
- `segments[].id`
- `segments[].faceCount`
- 最终 GLB source node identity
- 每个最终 GLB primitive 的 `faceLabels[]`
- provider artifact SHA / lineage

**禁止做法**：

- 只看总 face count 相同就假设 face order 没变；
- 在 AgentScape 侧按颜色反推 face IDs；
- 依赖不稳定 node 名称；
- 让 Adapter 猜 primitive 对应关系。

**验收**：

1. provider 导出的最终 GLB 被重新解析；
2. 每个 primitive triangle count 与对应 `faceLabels.length` 相等；
3. 所有 label 都属于声明的 segment；
4. segment faceCount 与 labels 实际统计一致；
5. 用冻结 fixture 送入 AgentScape `SegmentationEvidencePass + SegmentMaterializePass` 后 `materialization.status=materialized`；
6. 编译后 geometry/bounds 不发生数量级漂移。

### AFF-03：GraspGen raw 6-DoF inference

> **状态：VERIFIED（2026-08-24）**。实现已进入 `modal-build@69c910c`；真实 L40S/production URDF 已输出 top-20 finite 6-DoF grasps，并通过 rotation/score validation。

**目的**：先验证 GraspGen 模型自身，不依赖 GPT 语义筛选。

**仓库**：`modal-build`

**主要文件**：

- `modal_build/embodiedgen_graspgen_weights.py`
- `runtime/embodiedgen_affordance_l40s.py`
- `tests/test_embodiedgen_graspgen_weights.py`
- `tests/test_embodiedgen_affordance_runtime.py`

**输入**：成功 production Job 的 URDF bundle。

**坐标要求**：必须解析 URDF visual/collision mesh、`origin xyz/rpy` 与 `scale`。禁止直接假设 `sample_00.obj` 已经处在 GraspGen 所需 link frame。

**输出建议**：

- `affordance/raw_grasps.franka.v1.json`
- 可选 compact binary/Numpy artifact
- `affordance/graspgen_validation_report.json`

每个 candidate 至少记录：

- 4×4 pose / translation + rotation；
- discriminator score；
- gripper profile；
- source coordinate frame；
- source mesh/URDF SHA；
- model revision；
- generator/discriminator SHA；
- inference seed/options。

**验收**：

1. NVIDIA L40S；
2. runtime 无 `nvcc` / 无动态 CUDA build；
3. 权重离线加载且 SHA 固定；
4. 至少生成一个 finite pose；
5. rotation 合法、translation finite、score finite；
6. 输出重复运行在固定 seed 下满足定义好的 deterministic tolerance；
7. 不把 raw grasp candidate 写成 `pickup verified=true`。

### AFF-04：独立 semantic worker

状态：**GATED**。

前置：提供独立 GPT Secret/endpoint，不复用 build/release Secret。

设计要求：

- CPU/network worker，与 L40S P3-SAM / GraspGen 镜像隔离；
- 输入必须是 immutable segmentation artifact；
- 输出 `part_semantics.v1.json`，不原地覆盖 source；
- prompt/template/model/revision/request-id 可追溯，但 Secret 永不进入 artifact；
- semantic labels 只是 proposal，不生成 joint type/axis/limits；
- GPT failure 可独立 retry，不重跑 P3-SAM。

### AFF-05：SAPIEN grasp evaluation

前置：AFF-03 raw grasp + AFF-04 semantic。

必须把三种 evidence 分开：

- `raw_grasp`
- `semantic_selected_grasp`
- `sapien_validated_grasp`

SAPIEN 通过只代表指定 simulation profile 下通过，不能升级为 AgentScape/Rapier 已验证。

### AFF-06：统一 Affordance Job / bundle contract

只有 AFF-02/03 至少完成后再落地 Job API，避免把错误 artifact schema 固化成外部协议。

首版 stage 建议：

```text
ingest
-> segment
-> grasp_raw
-> semantic (optional/gated profile)
-> grasp_eval (optional/gated profile)
-> finalize
```

Job 必须允许 `part-evidence-only` profile，这样 AgentScape 不需要等待 GPT/SAPIEN 才能消费 P3-SAM segmentation。

## 6. AgentScape 当前真实接点

当前 AgentScape 已经存在：

- `AssetCompiler.compile({ ..., partProposal, partSegmentation })`
- `SegmentationEvidencePass`
- `SegmentMaterializePass`
- `PartProposalPass`
- `CompileQualityPass`
- `assetAdmission()`

因此下一步不是新增第二套 Compiler，而是新增 **EmbodiedGen bundle → Compiler input** 的转换层。

当前 `src/adapters/EmbodiedGenAdapter.js` 是 legacy loose-payload adapter：它会直接生成 provisional Manifest、fallback box collider，并根据 provider affordance 字符串添加 action。它不适合作为新的 enriched bundle 主路径。

新的 bundle 主路径必须：

1. 先验证 bundle/artifact/hash；
2. 下载/取得 GLB bytes；
3. 把 P3-SAM evidence 转成 `partSegmentation`；
4. semantic 完成后才生成 `partProposal.semantic`；
5. raw/SAPIEN grasps 保存在 provider evidence；
6. 调 `AssetCompiler.compile()`；
7. 由 Compiler / Admission 决定 ready/provisional/rejected；
8. 不直接构造“已验证 Manifest”。

详细任务见 `../group-3-agentscape/06-embodiedgen-evidence-bridge-execution.md`。

## 7. 文档同步协议

以后每次 `modal-build` 或 AgentScape 有以下事件，本文件都要同步更新：

- 新 commit 合入；
- 新 release/tag；
- 真实 Modal canary 成功/失败；
- artifact schema 改变；
- AgentScape Compiler contract 改变；
- 新 Secret / external dependency 变成可用或失效；
- 原 blocker 被消除或出现新 blocker。

每次更新至少写四件事：

1. **事实**：发生了什么；
2. **证据**：commit / test / report / release / job；
3. **影响**：哪些任务从 PLANNED → IMPLEMENTED → VERIFIED，或变成 GATED；
4. **下一任务**：文件、输入、输出、验收标准。

禁止使用没有证据来源的“基本完成”“应该可用”“大概率没问题”。

## 8. 当前立即执行顺序

AFF-02、AFF-03 与 AgentScape E-01 core 已解锁并验证。下一阶段按当前价值/依赖关系：

1. **AFF-06 part-evidence-only Job/API**：先把已验证 `segment + grasp_raw + finalize` 纳入统一异步 Job/stage contract，不等待 GPT；
2. **AS-EG-05 frozen fixture E2E**：把真实 provider contract 缩成脱敏、体积受控 fixture，锁住 BundleAdapter→Compiler→Admission 回归；
3. **AFF-04 semantic worker contract**：先实现 Secret/profile/prompt/provenance 边界；真实 E2E 仍受 GPT Secret Gate；
4. 配置 GPT Secret 后执行 AFF-04 真实 semantic canary；
5. **AFF-05 SAPIEN grasp evaluation**：只消费 raw + semantic-selected grasp，不覆盖原始 evidence；
6. **AS-EG-06 semantic/grasp 增量接入**：semantic / raw / SAPIEN evidence 分层进入 AgentScape；
7. 最后把完整 Affordance profile 接入 capability registry、Connector/Job transport 与 UI；
8. 任一 provider evidence 都不得越级变成 AgentScape runtime truth。
