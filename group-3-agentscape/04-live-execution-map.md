# AgentScape Live Execution Map

> 这是 **AgentScape 当前实现状态与下一步任务的动态执行文档**。它不替代 `00-master-roadmap.md`、`01-AgentScape-integration-plan.md` 或跨项目 contract；稳定文档回答“最终系统要长什么样”，本文件回答“当前代码已经做到哪里、下一项具体工作是什么、谁可以做、依赖什么、如何验收”。
>
> 状态快照：**2026-08-25 01:54 +08:00**。任何实施前必须重新读取 AgentScape `git status`、HEAD、分支、stash 与 CodeGraph；不要把本文件里的 commit/status 当永久事实。

## 1. 为什么需要这份 Live Map

AgentScape 已经进入多条并行工作流：Runtime truth 修复、1.35 WorldRevision、Backend productization、Provider/Connector/Artifact 链，以及未来的双生成策略。如果仍只用“AS-01～AS-19”表示进度，会产生四类误判：

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
                           AgentScape 当前代码事实
                 ┌─────────────────────────────────┐
                 │ Runtime / Physics / Interaction │
                 │ Generated World / Retry         │
                 │ Provider/Connector/Job/Artifact (merged)│
                 │ 1.35 WorldRevision stash        │
                 │ Backend handoff stash           │
                 └───────────────┬─────────────────┘
                                 │
                                 │ Git / CodeGraph / Tests
                                 v
                 ┌─────────────────────────────────┐
                 │      LIVE IMPLEMENTATION MAP    │
                 │ merged / branch / dirty / stash │
                 │ dependency / gate / acceptance  │
                 └───────────────┬─────────────────┘
                                 │
                                 v
        ┌─────────────────────────────────────────────────────┐
        │                AgentScape-plan                      │
        │                                                     │
        │ Architecture Contracts  -> stable target            │
        │ Concrete Work Slices    -> next executable tasks    │
        │ Dependency Gates        -> when work may start       │
        │ Acceptance Evidence     -> what counts as done       │
        │ Live Status             -> what is true right now    │
        └────────────────────────┬────────────────────────────┘
                                 │
                ┌────────────────┴────────────────┐
                │                                 │
                v                                 v
      Runtime / Truth Track               Generation Platform Track
  ┌──────────────────────────┐        ┌────────────────────────────┐
  │ placement stability      │        │ Provider Registry          │
  │ carry state              │        │ Connector                  │
  │ articulation truth       │        │ Async Job                  │
  │ agent sequencing         │        │ Artifact Importer          │
  └────────────┬─────────────┘        └──────────────┬─────────────┘
               │                                      │
               └──────────────────┬───────────────────┘
                                  v
                         1.35 Integration Gate
                     ┌────────────────────────┐
                     │ Runtime truth stable   │
                     │ Provider contract      │
                     │ WorldRevision          │
                     │ Backend policy         │
                     │ Full regression        │
                     └───────────┬────────────┘
                                 v
                     Providerized Generated World
                                 │
                  ┌──────────────┴──────────────┐
                  v                             v
          Strategy A                       Strategy B
        2D -> SAM -> 3D                  EmbodiedGen
                  \                         /
                   \                       /
                    v                     v
                      Compiler / Admission
                              │
                              v
                         WorldSpec v2
                              │
                              v
                  Deterministic World Runtime
                              │
                              v
                      Verified World Task
```

当前工程阶段定义为：

> **Late Core / Early Productization -> Providerized Generated World Foundation**

Runtime 的主要问题已经从“有没有能力”转向“能力真值是否稳定、失败是否可恢复”；生成侧的主要问题已经从“能不能请求 generator”转向“Provider/Job/Artifact 是否具备正式产品契约”。

## 3. 2026-08-25 当前实现账本

| Slice | 当前状态 | Git/实现身份 | 下一 Gate |
|---|---|---|---|
| R-01 Runtime truth stabilization | `MERGED` | 当前 `main@d28980d` 已包含此前 Runtime truth 修复 | 继续按独立 Runtime track 处理剩余 correctness |
| P-01 Provider Registry | `MERGED` | `0684af0 feat: add provider capability registry` | 已解锁 Connector/Job/Artifact |
| W-01 WorldSpec Revision | `STASH_PROTOTYPE` | 历史 stash prototype；未作为当前正式能力 | 独立恢复/重新验证 |
| B-01 Backend handoff | `STASH_PROTOTYPE` | 历史 stash prototype；未作为当前正式能力 | CORS/production boundary 独立收口 |
| R-02 Mutation atomicity | `PLANNED` | 当前 Live Map 仍无新的 merged fix 证据 | 独立 Runtime Gate |
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

2026-08-25 01:54 +08:00 重新读取后的代码事实：

- `main == origin/main == df9f9c1`；AS-09/AS-10A 均已推送到共享 main；
- HEAD：`df9f9c1 feat: add generation job center`；
- `0684af0 / fefa495 / 47470c0 / 6a08f31 / 140acd9 / ac3f996 / 09e63cc / 162a101 / a4b3297 / 571c747 / 5202f6f / 51bf326 / d28980d / 9523ecc` 均已确认在当前本地 main ancestry 中；
- 因此 Provider / Connector / Async Job / Artifact / single-asset Compiler→Admission 基础层均已进入 `main`，不应再标为 `PLANNED`；
- AS-09 `9523ecc` 已进入并推送共享 main：`GenerationOrchestrator`、Runtime generation 接线、7 个 Agent skills、generation policy/status 语义以及两组 generation tests；
- AS-09 在 `9523ecc` 上重新验证：完整 `npm test` 为 `136 files / 590 tests PASS`；`npm run assets:validate` PASS；`npm run build` exit code 0；此前 generation/Provider/Connector/Job/Artifact/Compiler 扩大回归为 `20 files / 192 tests PASS`；
- `9523ecc` 已进入共享 `origin/main`，AS-09 为 `MERGED`；其真值边界继续由后续 AS-10 UI 消费而不重写。

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
- 当前完整仓库验证已包含该基础层：`138 files / 598 tests PASS`，production build exit code 0。

### 3.3 1.35 Constrained WorldSpec Revision

状态：**`STASH_PROTOTYPE`**。

已确认 stash 中存在 `WorldRevision.js`、测试及 `ToolCallingAgent`/core skill 接线。真实控制流是：

```text
world pipeline rejected
        |
        v
restore(before)
        |
        v
bounded retry possible ?
    |              |
   yes             no
    |              |
    v              v
retry plan   buildWorldRevisionProposal
                   |
                   v
          world-rejected.revision
                   |
                   v
                Planner
```

它保持：

- `automatic = false`；
- `autoApply = false`；
- Runtime 只提供“哪个 constraint 有证据可修改”的 revision proposal；
- 不允许 Runtime 偷偷放宽用户 WorldSpec。

正式化前不能被其它任务当作已存在 API。

### 3.4 Backend 1.35 handoff

状态：**`STASH_PROTOTYPE`**。

已有 Agent Gateway / Asset Compiler production boundary、Docker、health、structured errors、timeout/CORS 等实现，但与当前 main 存在一个必须先决策的 contract 冲突：

```text
current main empty allowlist
        -> browser origins permissive

backend stash empty allowlist
        -> non-local browser origins denied
```

在 1.35 集成前必须把“dev permissive / prod explicit allowlist”变成显式 policy，不能让 empty-list 同时拥有两种相反含义。

### 3.5 Runtime mutation atomicity

状态：**`PLANNED`**，暂不与当前 Runtime WIP 并发修改。

已通过最小运行实验复现：`WorldRuntime.mutate()` 中 operation 先改变状态、随后抛异常时，`CommandHistory.cancel()` 只清理 pending history，不恢复 before snapshot；实验观察到 `rollbackObserved=false`。

这不代表所有高级 pipeline 都不会回滚——部分功能已有局部清理或显式 `restore(before)`——但通用 mutation contract 当前不是 exception-atomic。

建议将它作为 **1.35 Runtime Correctness Gate**，而不是继续隐藏在局部 caller 的补偿逻辑里。

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

**状态：READY AFTER R-01 + mutation contract decision。**

把 stash prototype 恢复成独立 feature commit，保持 proposal-only；新增 revision evidence contract 和 stale evidence/replan tests。

### B-01 Formalize Backend Handoff

**状态：READY AFTER CORS ADR。**

把 backend stash 拆成可审查 commit：Gateway errors/timeouts、health protocol、Docker、Compiler CORS；先写 CORS ADR 再合代码。

### R-02 WorldRuntime Mutation Atomicity

**状态：BLOCKED BY R-01 CURRENT OWNERSHIP。**

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

此任务必须有最小 reproducer + Runtime regression，不接受只在 caller 再加一个 try/catch workaround。

## 6. 并行工作 ownership 规则

为了允许多个 AI 同时推进，按“高 blast-radius Runtime / 低 blast-radius platform boundary”分轨：

| Track | 主要 ownership | 默认禁止并发修改 |
|---|---|---|
| Runtime Truth | Physics / Interaction / Spatial / ToolCallingAgent / sequencing | Provider/Connector AI 不改这些文件 |
| Provider Platform | `src/providers`、Asset gateway/library 最小接线 | 不改 Physics/Interaction |
| Connector/Job | 新 connector/job modules + contract tests | 不直接改 WorldRuntime |
| Artifact | artifact/import/storage modules | 不绕过 Compiler/Admission |
| World Revision | WorldSpec/WorldRetry/Planner proposal | 不修改 provider credential/job transport |
| Backend | gateway/service deployment boundary | 不改变 Runtime truth contract |

若一个任务必须跨 ownership，先在 Live Map 里标记冲突，再串行合并，不允许两个 agent 同时修改同一高风险文件后再靠人工猜 merge。

## 7. 阶段 Gate

### Gate L0：Runtime Truth Stable

- R-01 merged；
- 当前 dirty Runtime WIP 清零；
- regression baseline 明确；
- mutation atomicity 是否作为 1.35 blocker 已决策。

### Gate L1：Provider Contract Stable

- P-01 merged；
- provider/capability descriptor schema 稳定；
- capability availability/health/operation identity tests；
- legacy generator 仍兼容。

### Gate L2：Connector Session Stable

**状态：MERGED FOUNDATION。**

- C-01/C-02 已进入 main；
- credential 不进入 browser；
- session scope/version/revoke/expiry 完整；
- Connector capability -> ProviderRegistry normalization 完成。

### Gate L3：Async Job Truth Stable

**状态：MERGED FOUNDATION。**

- J-01/J-02 已进入 main；
- local Job identity 成为用户可见主身份；
- provider remote ID 只是 location/provenance；
- pending/provider_succeeded 不冒充 asset-ready。

### Gate L4：Artifact Truth Stable

**状态：MERGED FOUNDATION。**

- A-01/A-02 已进入 main；
- bytes/hash/role/lineage/locations；
- signed URL 不成为持久 identity；
- malformed/oversize/corrupt fail-closed。

### Gate L5：Single Asset End-to-End

**状态：MERGED。**

- generic modal-3D 或 EmbodiedGen 单资产；
- Job -> Artifact -> Compiler -> Admission；
- `asset-ready/provisional/rejected` 真实传播。

### Gate L6：Dual Strategy

- Strategy A：2D -> SAM -> 3D；
- Strategy B：EmbodiedGen Text -> bundle；
- 显式选择，不静默双跑/重复计费；
- fallback 创建 linked request。

### Gate L7：Generated World v2

- Prompt -> WorldSpec v2；
- reuse-first；
- multiple async assets；
- deterministic compose；
- rejected world rollback；
- provider artifacts 保留，不因 world rollback 删除。

### Gate L8：Environment / Room / Offline

- background/environment bundle；
- navigation rebuild；
- room feasibility Gate；
- scene serialize only compiled/artifact identity；
- offline restore。

## 8. 未来阶段优先级

### Phase 1：1.35 Correctness + Provider Foundation

R-01 -> P-01 -> W-01/B-01/R-02 按 ownership 串行收口。

### Phase 2：Providerized Async Asset Generation

**基础层已 MERGED：** C-01 -> C-02 -> J-01 -> J-02 -> A-01/A-02。后续不重复实现该链。

### Phase 3：Single Asset Production Loop

**核心链已 MERGED：** AS-05/06/08 已提供 artifact -> compiler evidence -> admission；当前 AS-09 把 Job/Artifact 正式接到 Agent-visible orchestration。

### Phase 4：Agent-visible Generation

AS-09 已 `MERGED`（`9523ecc`）；AS-10A Job Center core 已 `MERGED`（`df9f9c1`）。当前 UI 已覆盖 Connector/Job/Artifact/Compiler/Admission 的安全主链；下一项是 AS-10B richer schema/model/workflow UX + 真实 Connector product E2E。Agent/UI 始终不直接拿 provider 私有 API。

### Phase 5：Generated World v2

AS-11/12/13：Prompt -> WorldSpec v2、async fan-out、EmbodiedGen layout proposal；WorldSpec 仍由 Runtime compose/verify。

### Phase 6：Environment / Persistence / Hardening

AS-14～19：environment/room、offline restore、安全、observability、fault injection、E2E。

自动语义、自动 Joint/Target 推断、复杂 grasp/manipulation、Multi-Agent 都应在上述生成供应链与真值层稳定后再提高优先级；否则会扩大“provider evidence 被误升为 Runtime truth”的风险。

## 9. Live 更新协议

每次 AgentScape 发生有意义变化，按以下步骤更新本文件：

1. 读取 `git status --short --branch`；
2. 读取最新 `git log`，确认 main/branch/stash 身份；
3. dirty worktree 只标 `DIRTY_WIP`，不写“完成”；
4. 必要时 `codegraph sync`，再看关键 symbol/impact；
5. 记录测试证据，只有 `exit_code=0` 才写 PASS；
6. production build timeout 与失败分开记录；
7. branch 合并后把 `COMMITTED_NOT_MERGED` 改成 `MERGED`；
8. stash 只有工程化为正式 commit 后才能升级状态；
9. 任务完成后必须写“下一依赖 Gate 是否解锁”；
10. 稳定架构/contract 变化才回写 `01/02/03` 和 master roadmap；纯实施进度只更新 Live Map。

## 10. 下一次同步时优先检查

按当前状态，下一次读取 AgentScape 时优先回答：

1. AS-10A 的 Job Center 是否已在真实 Connector 进程上完成 pair → capability → submit → restart/reconcile → artifact → compile 的浏览器产品 E2E？
2. Provider capability contract 是否已经真实提供 model/workflow/profile/optionsSchema 元数据；若没有，UI 不得自行硬编码 provider 私有 catalog。
3. 哪些 JSON Schema 子集值得在 AS-10B 生成结构化表单，哪些仍应保留 JSON fallback？
4. cost confirmation 是否需要按 provider/capability cost class 增加更细 policy，而不是只有统一的逐次确认？
5. EmbodiedGen multi-artifact bundle 是否需要在 Job Center 中增加 artifact role selector，再进入 BundleAdapter，而不是默认优先 GLB？
6. real Connector restart 后 local Job identity / idempotency / event cursor 是否都能无重复付费恢复？
7. AS-10B 完成后是否具备进入 AS-11 Prompt→WorldSpec Planner 的证据；不得因 UI 可见就跳过 asset/world truth Gate。

这些问题决定下一项任务，不按日期机械推进 AS 编号。
