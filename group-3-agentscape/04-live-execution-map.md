# AgentScape Live Execution Map

> 这是 **AgentScape 当前实现状态与下一步任务的动态执行文档**。当前 HEAD/commit/tests/实现状态以本文为准；未来目标架构、World IR、可替换物理后端、G0～G8 与多 AI ownership 以 [`07-agent-native-world-architecture-replan.md`](./07-agent-native-world-architecture-replan.md) 为权威。旧 `01` 的 AS-11～19 与本文旧 L6～L8 不再作为未来线性执行顺序。
>
> 状态快照：**2026-08-26 +08:00**。本次复核：AgentScape `main == origin/main == ca8cab7`，`git stash list` 为空；任何实施前仍必须重新读取 `git status`、HEAD、分支、stash 与关键代码。

## 1. 为什么需要这份 Live Map

AgentScape 已经从 Provider/Connector 基础建设阶段进入 **World Compilation Architecture / 世界编译架构** 阶段。当前并行工作应围绕 Runtime correctness、World IR、Interaction/Rule、PhysicsBackend abstraction、Verification/Repair 与 Provider product E2E 展开。如果仍只用“AS-01～AS-19”表示进度，会产生四类误判：

1. **文档已有 ≠ main 已实现**；
2. **branch 已提交 ≠ main 已合并**；
3. **stash prototype ≠ 正式 capability**；
4. **provider success ≠ asset/world verified**。

因此本文件强制把代码状态分成以下级别：

| 状态 | 定义 | 是否可作为后续依赖 |
|---|---|---|
| `MERGED` | 已进入当前 `main`，且有明确回归证据 | 可以 |
| `COMMITTED_NOT_MERGED` | 独立 branch/commit 已完成，但未进入 main | 只能作为候选依赖 |
| `DIRTY_WIP` | 当前工作树未提交实现 | 不可以 |
| `STASH_PROTOTYPE` | stash 中存在有价值实现/测试 | 不可以，必须工程化后再依赖 |
| `PLANNED` | contract/任务已定义但无代码 | 不可以 |
| `BLOCKED` | 前置 contract、实现或验证不满足 | 不可以 |

任何“完成”必须同时给出 **代码位置 + Git 身份 + 测试/构建证据**。

## 2. 当前实时架构阶段

```text
                         AgentScape current / 当前
                                  │
        ┌─────────────────────────┼──────────────────────────┐
        │                         │                          │
        ▼                         ▼                          ▼
Runtime Truth               Asset Truth              Generation Support
运行时真值                  资产真值                 生成支撑
Physics/Interaction         Compiler/Admission       Provider/Job/Artifact
        │                         │                          │
        └─────────────────────────┼──────────────────────────┘
                                  ▼
                        main@ca8cab7 / v1.34.2
                                  │
                                  ▼
                  ┌────────────────────────────────┐
                  │ New Architecture Gates / 新门 │
                  │                                │
                  │ G0 Runtime correctness         │
                  │ G1 World IR                    │
                  │ G2 Physics + Behavior Contract │
                  │ G3 Executable Behavior         │
                  │ G4 Canonical World Compilation │
                  │ G5 Semantic Asset Automation   │
                  │ G6 World Acceptance + Repair   │
                  └───────────────┬────────────────┘
                                  │
                                  ▼
                       Verified World System
                         已验证世界系统
                                  │
                     ┌────────────┴────────────┐
                     ▼                         ▼
              G7 Multi-backend          G8 Scale / Rich World
              多物理后端                规模化 / 丰富世界
```

当前工程阶段重新定义为：

> **Late Core / Early World-Compiler Productization / 核心后期、世界编译器产品化早期。**

Provider/Connector/Job/Artifact 基础已经不再是所有工作的总 blocker。当前最重要的是把已经很强的 Runtime/Compiler/Verification 能力收敛成稳定的 **World IR + Behavior Contract + PhysicsBackend Contract + World-level Acceptance**。

## 3. 2026-08-25 当前实现账本

| Slice | 当前状态 | Git/实现身份 | 下一 Gate |
|---|---|---|---|
| R-01 Runtime truth stabilization | `MERGED` | `d28980d` 已在当前 `main@7bbe4b2` ancestry 中 | Runtime truth baseline 保持 |
| P-01 Provider Registry | `MERGED` | `0684af0 feat: add provider capability registry` | 已解锁 Connector/Job/Artifact |
| W-01 WorldSpec Revision | `STALE_HISTORICAL_RECORD` | 当前无 stash、无 `WorldRevision.js` | 由 G1/G4 的 IR revision 重新实现 |
| B-01 Backend handoff | `STALE_HISTORICAL_RECORD` | 当前无可恢复 backend stash | 如仍需要，Support Plane 重新开 task spec |
| R-02 / R-ATOMIC-01 Mutation atomicity | `MERGED` | `3a956dc`：partial throw → restore(before)；rollback failure fail-closed；snapshot failure 解锁 | G0 已闭合 |
| C-01 Connector Pairing | `MERGED` | `fefa495 feat: add scoped connector pairing sessions` | 已解锁 C-02/J-01 |
| C-02 Capability Adapter | `MERGED` | `47470c0 feat: add connector capability snapshots` | 已解锁 J-01 |
| J-01 Async Job | `MERGED` | `6a08f31 feat: add async generation job projection` | 已解锁 J-02/A-01 |
| J-02 Reconcile/Recovery | `MERGED` | `140acd9 feat: add job reconcile and event recovery` | 已解锁 Agent-visible recovery |
| A-01 Artifact Descriptor | `MERGED` | `ac3f996 feat: add artifact identity and lineage registry` | 已解锁 A-02 |
| A-02 Artifact Importer | `MERGED` | `09e63cc feat: add streaming artifact integrity importer` | 已解锁 verified asset pipeline |
| AS-05 Single Asset Pipeline | `MERGED` | `162a101 feat: add verified artifact asset production loop` | 已进入 Compiler/Admission truth |
| AS-06/08 Evidence + Admission | `MERGED` | `a4b3297` / `571c747` / `5202f6f` / `d28980d` | provider evidence 保持非 Runtime truth |
| E-01 EmbodiedGen part evidence bridge | `MERGED` | `671e1ac` + `51bf326` + `d28980d` | production provider transport / SAPIEN provider Gate 继续独立推进 |
| AS-09 Agent-visible async generation | `MERGED` | `9523ecc feat: add agent-visible async generation orchestration` | 已进入共享 main |
| AS-10A Job Center core | `MERGED` | `df9f9c1 feat: add generation job center` | AS-10B richer schema/model/workflow UX + real Connector product E2E |

### 3.1 AgentScape main

2026-08-25 21:57 +08:00 重新读取后的代码事实：

- `main == origin/main == 7bbe4b2`；AS-09/AS-10A、依赖升级、CI test-deps、R-ATOMIC-01 均已推送到共享 main；
- HEAD：`7bbe4b2 merge: make world mutations exception atomic`；
- `0684af0 / fefa495 / 47470c0 / 6a08f31 / 140acd9 / ac3f996 / 09e63cc / 162a101 / a4b3297 / 571c747 / 5202f6f / 51bf326 / d28980d / 9523ecc` 均已确认在当前本地 main ancestry 中；
- 因此 Provider / Connector / Async Job / Artifact / single-asset Compiler→Admission 基础层均已进入 `main`，不应再标为 `PLANNED`；
- AS-09 `9523ecc` 已进入并推送共享 main：`GenerationOrchestrator`、Runtime generation 接线、7 个 Agent skills、generation policy/status 语义以及两组 generation tests；
- AS-09 在 `9523ecc` 上重新验证：完整 `npm test` 为 `136 files / 590 tests PASS`；`npm run assets:validate` PASS；`npm run build` exit code 0；此前 generation/Provider/Connector/Job/Artifact/Compiler 扩大回归为 `20 files / 192 tests PASS`；
- `9523ecc` 已进入共享 `origin/main`，AS-09 为 `MERGED`；其真值边界继续由后续 AS-10 UI 消费而不重写。
- 当前已用 `npm ci` 从 lockfile 重装并执行 `npm run check`：`139 files / 607 tests PASS`，Vite 8 production build exit code 0；`npm audit` 为 0 vulnerabilities。Python Asset Compiler 在 Python 3.11 + `trimesh 5.0.0` + `httpx2 2.12.0` 下 `7/7` unittest PASS。
- `git stash list` 当前为空；旧 WorldRevision/backend stash 记录已降级为历史信息。

当前生成主链已达到：

```text
Provider/Capability
        -> local Generation Job
        -> generation-pending / provider-succeeded
        -> Artifact descriptor/import/hash verification
        -> VerifiedArtifactAssetPipeline
        -> AssetCompiler
        -> Admission
        -> asset-ready / asset-provisional / asset-rejected
```

真值边界继续强制保持：`provider-succeeded != artifact-imported != asset-ready != world-ready`。

### 3.2 AS-01 Provider Registry

状态：**`MERGED`**。

- 当前 main 身份：`0684af0 feat: add provider capability registry`；
- Provider identity/version/health/status/capability discovery/execution binding/result consumer 已正式进入主线；
- Connector capability snapshot 已通过 C-02 normalize 后写入 Registry；
- 后续任务不得重新实现第二套 Provider Registry，也不得把 provider 私有 schema 直接交给 Agent/UI；
- 当前完整仓库验证已包含该基础层：`139 files / 607 tests PASS`，production build exit code 0。

### 3.3 Constrained World Revision 历史记录

状态：**`STALE_HISTORICAL_RECORD`**，不可作为代码依赖。

旧 Live Map 曾记录一个 `WorldRevision.js` stash prototype；本次复核 `git stash list` 为空，当前代码树也没有 `WorldRevision.js`。因此不能再描述成“可以恢复的 stash prototype”。

仍然有效的是当时验证出的设计原则：

```text
world rejected / 世界被拒绝
        ↓
restore before / 恢复前态
        ↓
finding evidence / 问题证据
        ↓
constrained IR revision proposal / 受约束 IR 修订提议
        ↓
changed-plan gate / 变更计划门
        ↓
canonical recompile / 标准重编译
```

未来正式实现由 `07` 的 **G1 World IR + G4 Canonical World Compilation** 重新定义，不依赖不存在的 stash。Runtime 仍不得偷偷放宽用户约束或直接把 finding patch 成永久 live-world truth。

### 3.4 Backend handoff 历史记录

状态：**`STALE_HISTORICAL_RECORD`**，不可作为可恢复实现。

旧 Live Map 曾记录 Gateway / Compiler production boundary 的 backend stash，但当前 AgentScape `git stash list` 为空，当前树也没有对应 Docker handoff 成套实现。因此后续若仍需要该 productization，必须基于当前代码重新形成独立 task spec，而不是写“恢复 backend stash”。

仍然有效的 contract：dev permissive / prod explicit allowlist 必须是显式 policy；空 allowlist 不能同时拥有相反语义。

### 3.5 Runtime mutation atomicity

状态：**`PLANNED`**，暂不与当前 Runtime WIP 并发修改。

已通过最小运行实验复现：`WorldRuntime.mutate()` 中 operation 先改变状态、随后抛异常时，`CommandHistory.cancel()` 只清理 pending history，不恢复 before snapshot；实验观察到 `rollbackObserved=false`。

这不代表所有高级 pipeline 都不会回滚——部分功能已有局部清理或显式 `restore(before)`——但通用 mutation contract 当前不是 exception-atomic。

它现在正式归入新计划 **G0 Runtime Truth & Baseline Freeze**，任务号 `R-ATOMIC-01`；不再以版本号 1.35 作为架构前置。

### 3.6 EmbodiedGen Affordance provider evidence

状态：**provider semantic profile VERIFIED / AgentScape semantic bridge VERIFIED**。

2026-08-24 已同步到远端 main/master 的真实事实：

- `modal-build@eda84b7`：`part-evidence-only` 与 `semantic-evidence-v1` 两个 derived Job profiles 均已真实 E2E；
- P3-SAM：真实 production Job 50,000 faces 全覆盖，4 parts；
- compiler-native evidence：绑定 primary GLB SHA、sourceNode=`geometry_0`、primitive labels 50,000，provider 端显式验证 OBJ↔GLB vertex/triangle identity；
- GraspGen：真实 production URDF→top-20 raw grasps，score/rotation 均 finite，artifact 明确 `evidence_level=raw`；
- `AgentScape@671e1ac`：新增 `EmbodiedGenBundleAdapter`、providerEvidence provenance、provider-aware CompileQuality/Admission reasons；
- 真实 Bundle→Compiler→Admission：`materialized`、coverage=1、4 parts、hard=0、final=`provisional`；
- raw grasp 未越权提升为 pickup truth；
- provider semantic 已由 `modal-build@be697af` 真实 E2E；`eda84b7` 已把它纳入完整 semantic profile，最新 canary 为 5 parts；SAPIEN 仍未 VERIFIED。
- derived Job `job-a2595a4645f6454cb9d4dbc2b0dff692` 已成功产出 bundle v1，三阶段约 `37.6s / 19.2s / 3.9s`；
- proxy-auth HTTP route 已部署，但真实带认证 POST 尚未执行。
- `AgentScape@d28980d` 已验证真实 5-part semantic bundle：`partSemantics=provider-verified`、coverage=1、5 parts materialized、`promoted=[]`、manifest actions 仅 `move`、final=`provisional`。

因此 E-01 的 core contract 已通过。剩余 Gate 是：把真实 contract 冻结成脱敏 fixture，并接入正式 Artifact/Job transport；AgentScape 仍不得通过 provider semantic/grasp 直接构造未经 Runtime 验证的 joint/action truth。详细跨仓任务见 [`../group-2-embodiedgen/04-live-execution-state.md`](../group-2-embodiedgen/04-live-execution-state.md) 与 [`06-embodiedgen-evidence-bridge-execution.md`](./06-embodiedgen-evidence-bridge-execution.md)。

## 4. 下一阶段不是“继续加 Runtime primitive”

下一阶段的主目标是：

> **Providerized + Asynchronous + Artifact-addressed Generated World**

目标控制流：

```text
User / Agent intent
       |
       v
Capability Discovery
       |
       v
Provider Selection
       |
       v
Async Local Job
       |
       v
Provider Remote Job / Workflow
       |
       v
Artifact Descriptor + Hash + Lineage
       |
       v
Artifact Importer
       |
       v
Compiler Evidence Bridge
       |
       v
Asset Admission
       |
       v
WorldSpec / Composer
       |
       v
World Admission
       |
       v
Verified Runtime Task
```

最重要的 contract 不变：

```text
Provider succeeded
      !=
Artifact imported
      !=
Asset compiled
      !=
Asset ready
      !=
World ready
      !=
Task verified
```

## 5. 下一批具体任务：可直接分配给 AI 的切片

下面的切片比 AS-01～AS-19 更细。每项必须遵守“最小代码 ownership + 独立验收”。
正式交付给 AI/开发者时，应基于 [`05-execution-task-spec-template.md`](./05-execution-task-spec-template.md) 为该切片建立 Task Spec；Live Map 排顺序，Task Spec 固化执行 contract。

### R-01 Runtime truth stabilization

**状态：IN PROGRESS / 另一执行轨。**

范围：placement settle、carry truth、articulation truth、agent deterministic quick-task/replan regression。

禁止其它 track 同时大改：

- `InteractionSystem.js`；
- `SpatialSystem.js`；
- `ToolCallingAgent.js`；
- Runtime sequencing tests。

完成 Gate：

- dirty worktree 收口为 commit；
- 相关回归 PASS；
- 无 pending Runtime correctness regression。

### P-01 Provider Registry foundation

**状态：MERGED — `0684af0`。**

该 Gate 已完成，不再作为待执行任务。后续实现必须复用现有 Provider Registry，并继续保持：

- provider descriptors 不泄露 secret；
- stable operation ID；
- capability discovery 不依赖 provider-specific branch；
- disabled provider 不被误判 available；
- raw result 必须经过 registered consumer；
- provider evidence 不直接升级为 Runtime action/capability。

### C-01 Connector Pairing Contract（AS-02a）

**状态：MERGED — `fefa495`；当前 main 已包含 scoped Connector pairing。**

只定义/实现本地 Connector 会话边界，不碰 Async Job DB：

- pair request / approval；
- scoped session identity；
- Connector version；
- capability snapshot revision/hash；
- expiry / revoke；
- `connection_required`；
- origin/scope policy；
- 不允许 browser 看到 Modal/provider credential。

验收必须覆盖：wrong origin、expired token、revoked session、version mismatch、scope escalation。

### C-02 Capability Discovery Adapter（AS-02b）

**状态：MERGED — `47470c0`；Connector capability snapshot → ProviderRegistry normalization 已进入 main。**

Connector capability response -> AgentScape `ProviderRegistry` descriptor；必须是“normalize 后注册”，不允许 UI/AssetLibrary 直接消费 provider 私有 schema。

验收：

- unknown fields 保留 raw provenance 或忽略且可追踪；
- provider capability 暂时 unavailable 时 registry 正确降级；
- capability hash/revision 可用于 task/job provenance。

### J-01 Async Generation Job State Machine（AS-03a）

**状态：MERGED — `6a08f31`；本地 Generation Job identity/state 已进入 main。**

首版只实现本地 Job identity/state，不同时实现 UI Job Center。

建议状态至少区分：

```text
created
  -> submission_pending
  -> submitted
  -> running
  -> provider_succeeded
  -> importing_artifacts
  -> terminal

side states:
connection_required
cancel_requested
failed
```

关键原则：`provider_succeeded` 绝不能被命名成 `completed`，因为 Artifact/Compiler/Admission 尚未完成。

### J-02 Async Reconcile / Restart Recovery（AS-03b）

**状态：MERGED — `140acd9`；poll/event reconcile、restart recovery 基础层已进入 main。**

任务：本地进程重启后通过 provider identity + remote job ID reconcile，不依赖过期 signed URL。

验收：restart、duplicate poll/SSE、out-of-order event、cancel race、provider terminal but artifact expired。

### A-01 Artifact Descriptor Contract（AS-04a）

**状态：MERGED — `ac3f996`；Artifact identity/lineage/locations/lease contract 已进入 main。**

定义 opaque artifact ID、role、hash、MIME、bytes、locations、lineage、lease/retention；remote URL 只是 location，不是 artifact identity。

### A-02 Artifact Importer Bytes/Hash Gate（AS-04b）

**状态：MERGED — `09e63cc`；streaming bytes/hash/MIME gate 已进入 main。**

实现 streaming import、length/hash/MIME/budget、temporary location cleanup；禁止直接把 signed URL 持久化进 scene/world truth。

### E-01 EmbodiedGen Part Evidence Bridge

**状态：MERGED；A-01/A-02 transport 与 frozen fixture 均已进入 main，且已有验证证据。**

两段 core task 均已完成：

1. provider (`modal-build@69c910c`) 发布与 final GLB primitives 严格对齐的 `agentscape_part_segmentation.v1` artifact；
2. AgentScape (`671e1ac`) 新增 `EmbodiedGenBundleAdapter`，机械转换为现有 `AssetCompiler.compile({ partSegmentation, providerEvidence })` 输入。

真实 production Bundle 验收：

- `SegmentationEvidencePass.issues=[]`；
- `SegmentMaterializePass.materialization.status='materialized'`；
- coverage=1；
- provider/materialized part count 均为 4；
- final quality=`provisional`，hard findings=0；
- admission 明确包含 `PART_SEMANTICS_UNVERIFIED`、`PROVIDER_GRASP_RAW_ONLY`；
- semantic/grasp evidence 未越权提升为 joint/action truth；
- legacy `EmbodiedGenAdapter` 未修改。

AS-EG-05 base fixture 与 semantic fixture 都已完成；semantic bridge 已由 `d28980d` 验证。Artifact/Job 基础 transport 已进入 main；下一跨仓 Gate 聚焦真实 production provider transport 与 SAPIEN provider evidence，仍不得提升为 AgentScape Runtime grasp truth。详细文件级任务见 [`06-embodiedgen-evidence-bridge-execution.md`](./06-embodiedgen-evidence-bridge-execution.md)。

### AS-09 Agent-visible Async Generation Orchestration

**状态：MERGED — `9523ecc`；已进入当前本地 main，并在该 commit 身份上完成验证。**

当前本地实现已经完成首版 Agent-visible generation orchestration：

- 新增 `src/generation/GenerationOrchestrator.js`；
- Runtime 可选择配置 Connector，并将 generation service 注入 Skills；
- Agent 工具：`listGenerationProviders`、`listGenerationCapabilities`、`submitGenerationJob`、`getGenerationJob`、`cancelGenerationJob`、`importGenerationResult`、`generateAndCompileAsset`；
- submit/cancel/import 属于外部 Job/Artifact side effect，不使用 World history mutation；
- 相同 generation request 派生稳定 request hash/idempotency key，并优先复用本地 Job projection，避免重复付费提交；
- `provider-succeeded` 与 `artifact-imported` 在 Skill execution policy 中均保持 `unverified`；
- Artifact 必须通过现有 streaming bytes/hash/MIME gate 后才能进入 `VerifiedArtifactAssetPipeline`；
- 高层 orchestration 在 Job 未完成时直接返回 `generation-pending`，不会同步阻塞等待 Provider；
- Provider success 后仍必须经过 Artifact import → Compiler → Admission，最终才产生 `asset-ready/provisional/rejected`。

本地验收证据：

- generation E2E 证明 `generation-pending -> provider-succeeded -> artifact-imported -> asset-provisional`，且 provider success 时 AssetManager 中仍无该资产；
- 相同请求重复 submit 的测试证明 remote POST 只发生一次；
- secret-like generation payload 在 Skill execution/trace 之前 fail closed；
- generation/Provider/Connector/Job/Artifact/Compiler 扩大回归：`20 files / 192 tests PASS`；
- full suite：`136 files / 590 tests PASS`；
- `npm run assets:validate`：PASS；
- `npm run build`：exit code 0。

AS-09 merge/push Gate 已完成；其 UI 消费层已由 AS-10A Job Center core 接续。

### AS-10A Generation Job Center Core

**状态：MERGED — `df9f9c1`；已推送共享 `origin/main`。**

已完成第一版产品级 Job Center core：

- 工作台新增独立“生成”Tab，不与任务/对象面板混合；
- Connector endpoint 只保存 loopback URL；支持配对/恢复/撤销，并明确不在浏览器保存 provider Secret；
- UI 只消费 `GenerationOrchestrator` 的 sanitized control-plane view，不直接读取/复制 Job store 状态机；
- Provider/Capability browser 显示 normalized operation、input schema/type、profile、cost/duration class、auth/connection prerequisite；
- 外部 Job submit 必须逐次显式勾选“可能产生费用”确认，不跨 Job 持久化确认状态；
- Job list 展示 status/stage/progress/relations，可 refresh/reconcile、cancel；
- `provider-succeeded` 后才开放 Artifact import / compile；Provider success 本身不会开放“加入世界”；
- Artifact import 展示 integrity/hash/producer/lineage；
- Compiler/Admission 结果展示 `asset-ready/provisional/rejected` 与 admission reasons；
- “导入 Artifact”“编译/注册资产”“加入当前世界”“用于 WorldSpec”是四个分离动作；WorldSpec 动作保持 disabled，等待 AS-11；
- provider success + compiler rejected 会显式显示“不会提升为 asset-ready”；
- provisional asset 如进入当前世界仍通过现有 `spawnAsset` 编辑态语义，不冒充 world-ready。

实现边界：

- 新增 `src/generation/GenerationJobCenter.js`；
- `GenerationOrchestrator` 新增 safe connector status、cached Job list、reconcile、pair/revoke 控制面，并把 relations/model-independent capability metadata 交给 UI；
- `main.js` 只负责挂载 Job Center；动态 provider/job/artifact 字段全部通过 DOM `textContent` 渲染；
- 没有新增 provider credential persistence，没有把 signed URL/provider private response 暴露给 UI；
- 没有修改 Physics / Interaction / Navigation / World admission truth。

验收证据（`df9f9c1`）：

- Job Center 专项：`4 files / 17 tests PASS`；
- full suite：`138 files / 598 tests PASS`；
- `npm run assets:validate`：PASS；
- `npm run build`：exit code 0（仅保留既有 browser externalization/chunk-size warnings）。

AS-10 尚未完全宣告结束。下一 Gate 为 **AS-10B**：只有当 Provider contract 真正提供 richer model/workflow/schema metadata 时，再把当前 JSON fallback 升级成更强的 schema-driven fields/model/workflow picker，并补真实 Connector 进程级 product E2E。禁止 UI 自己虚构 provider model/workflow catalog。

### W-01 Formalize Constrained WorldSpec Revision

**状态：SUPERSEDED BY G1/G4。**

当前不存在可恢复 stash。该目标保留，但必须从 World IR revision contract 重新实现：proposal-only、evidence-linked、stale-evidence safe、changed-plan gated，并重新进入 canonical pipeline。

### B-01 Formalize Backend Handoff

**状态：DEFERRED / RE-SCOPE REQUIRED。**

当前不存在可恢复 backend stash。若产品部署仍需要 Gateway/health/Docker/CORS handoff，重新按 Support Plane task spec 定义，并先冻结 CORS/security policy。

### R-02 / R-ATOMIC-01 WorldRuntime Mutation Atomicity

**状态：MERGED — `3a956dc`，已进入 `main@7bbe4b2`。**

目标 contract：

```text
before snapshot
   |
operation partially mutates
   |
throws
   |
restore(before)
   |
no undo entry
mutationOwner cleared
scene/runtime truth == before
```

已完成：

- partial mutation 后 operation 抛错：`history.cancel()` 后 `restore(before)`；
- rollback 自身失败：显式 `WORLD_MUTATION_ROLLBACK_FAILED` + `AggregateError`，不冒充恢复成功；
- `snapshot()` 失败：`mutationOwner` 仍通过 outer `finally` 清空；
- 无 undo entry；
- 专项 mutation/history/serializer regression PASS；
- full suite：`139 files / 607 tests PASS`；production build PASS。

Editor 的 `beginMutation/commitMutation` 属于独立的手动 gizmo transaction 入口，不混入本次 async `mutate()` 修复提交；若后续需要 cancel/abort contract，单独开 editor transaction slice。

## 6. 并行工作 ownership 规则

未来 ownership 不再按“Provider/Connector/Artifact/Backend”作为主轴，而按 `07` 的五大核心 + 支撑层划分：

| Owner | 主要 ownership | 禁止越权 |
|---|---|---|
| AI-1 World IR / Planner | World IR schema、revision、planner output | 不改 Runtime/Physics truth |
| AI-2 Asset Compiler | Manifest、part/joint/collider evidence、Asset Admission | 不把 provider 标签直接升格 |
| AI-3 Interaction / Rules | capability、precondition/effect、state transition、rule graph | 不直接控制 solver |
| AI-4 Runtime / Physics | WorldRuntime mutation、PhysicsBackend、spatial/navigation runtime | 不让 UI/Planner 成第二 truth |
| AI-5 Verification / Repair | finding、verifier、acceptance、repair proposal | 不直接 patch IR/Runtime |
| AI-6 Provider / Generation | Connector、Provider、Job、Artifact、Job Center provider UX | 不改 core truth contract |
| AI-7 Human / Persistence / Content | editor、serializer、history、environment/demo | UI 不保存第二份执行状态 |
| AI-8 Integration Guardian | cross-contract tests、merge gate、compatibility | 不通过绕过 owner 解决冲突 |

高风险规则：`WorldRuntime` mutation 与 PhysicsBackend migration 由 AI-4 串行控制；跨 contract 修改先 proposal → owner → compatibility tests → AI-8 Gate。

## 7. 当前 Gate 视图

旧 L0～L5 作为 Provider foundation 历史仍成立；旧 L6～L8 已被新计划取代。当前执行看下面的 G Gate：

### G0 — Runtime Truth & Baseline Freeze / Runtime 真值与基线冻结

**状态：COMPLETE。**

- Provider/Job/Artifact/Asset foundation 已在 main；
- `WorldRuntime.mutate()` exception atomicity 已由 `3a956dc` 闭合并有 regression；
- partial mutation throw 可 restore before；rollback failure fail-closed；snapshot failure 不泄漏 owner；
- `npm ci` 后 full suite `139 files / 607 tests PASS`，production build PASS；
- architecture ownership / truth boundary 已在 `07` 冻结。

### G1 — World IR vNext Contract / 世界 IR 契约

**状态：COMPLETE — `281e02c` / `main@ca8cab7`.**

已完成：revision/provenance、PhysicsRequirement、capability/state intent、interaction/rule intent、acceptance、serialize/parse、legacy WorldSpec normalization；current executable pipeline 对未编译富语义 fail-closed；WorldRetry 保留 revision/provenance 并由 revised IR 生成 nextPlan。

### G2A — Physics Interface Parity / 物理接口等价抽象

**状态：NEXT CORE SLICE — dependency audit / contract RFC；Rapier parity implementation 保持独立 slice。**

目标是 `PhysicsBackend Contract → RapierAdapter`，第一阶段零主动行为变化，不接 Genesis/PhysX。

### G2B — Interaction & Rule Contract / 交互与规则契约

**状态：PLANNED，可与 G2A 并行。**

先 schema/contract，再做行为纵向切片。

### G3 — Executable Behavior Vertical Slice / 可执行行为纵向切片

**状态：BLOCKED BY G2B + Runtime mapping。**

首批建议 OPEN/CLOSE、PICKUP/PLACE、SWITCH。

### G4 — Planner + Canonical World Compilation / 世界规划与标准编译

**状态：BLOCKED BY G1/G2B/G3 contract。**

这是旧 AS-11/12 的替代：Prompt → strict World IR → asset/behavior/physics admission → compose → runtime → verify → constrained revision。

### G5 — Semantic Asset Automation / 语义资产自动化

**状态：PARTIALLY AVAILABLE / 可并行。**

现有 EmbodiedGen semantic evidence bridge 可继续提供 evidence；自动 joint/capability promotion 仍必须服从 Compiler/Verification Gate。

### G6 — World-level Acceptance & Local Repair / 世界级验收与局部修复

**状态：PLANNED。**

动作级验证已有强基础，世界级 acceptance + affected-IR repair 仍未形成完整 contract。

### G7 — Multi-backend Physics / 多物理后端

**状态：DEFERRED UNTIL G2A + G6。**

先 validation-only backend，再考虑第二 live backend；没有 coupling/snapshot/verification contract 不进入多后端 live truth。

### G8 — Scale / Environment / Persistence / Multi-Agent / 规模化

**状态：LATER。**

Environment/Room、large world、soft body、multi-agent 在核心编译链稳定后推进。

## 8. 当前可并行执行的任务

按 `07`，第一批建议：

```text
AI-1  IR-01        World IR Contract RFC + Compatibility Normalizer
AI-3  BEH-01       Capability / State / Rule Contract
AI-5  VER-01       Unified Finding + Acceptance Contract
AI-6  GEN-01       Real Connector Product E2E
AI-7  UX-01        IR/Finding/Verification observability requirements
AI-8  INT-01       Cross-contract test matrix
```

`R-ATOMIC-01` 已 merged。`PHY-01 PhysicsBackend Contract + Rapier Parity Adapter` 现在允许进入 dependency audit / contract RFC；实际 adapter 迁移仍需保持零主动 physics behavior change。

AS-10B 继续是 **conditional**：只有 Provider capability 真实提供 richer model/workflow/optionsSchema metadata 才做 schema-driven UI，不允许 UI 自行硬编码 provider catalog。

## 9. Live 更新协议

每次 AgentScape 有意义变化：

1. `git status --short --branch`；
2. `git log` 确认 HEAD；
3. `git stash list` 必须真实检查，stash 为空就不能继续写 `STASH_PROTOTYPE`；
4. 读取对应 owner 的核心 contract；
5. 只有 `exit_code=0` 才记录 PASS；
6. timeout 与 test failure 分开；
7. branch 合并后才写 `MERGED`；
8. 新 schema/architecture 变化同步 `07`；
9. Provider transport 变化同步 `02`；
10. 当前 commit/tests/next slice 只写 Live Map。

## 10. 下一次同步时优先检查

下一次读取 AgentScape 时优先回答：

1. `IR-01` 是否已经形成 World IR vNext contract，并兼容当前 WorldSpec normalize？
2. `PHY-01` dependency audit 是否已经枚举全部 Rapier-specific coupling 并冻结 backend-neutral query/snapshot contract？
3. `BEH-01` 是否把 capability/precondition/effect/state/verifier target 从散落逻辑收敛成 typed contract？
4. `VER-01` 是否统一 action/world finding、acceptance linkage 与 stale-evidence identity？
5. `BEH-01` / `VER-01` 是否能与 IR-01 contract 对齐而不产生第二份 state/truth？
6. real Connector 是否完成 pair → capability → submit → restart/reconcile → artifact → compile 产品 E2E？
7. Provider 是否真实提供 model/workflow/optionsSchema；若没有，AS-10B 保持 blocked/conditional。
8. 是否有任何 AI 新增第二份 Runtime/Physics/World truth；若有，优先阻断而不是继续叠功能。

这些问题决定下一项任务，不再按旧 AS-11→19 或 L6→L8 机械推进。
