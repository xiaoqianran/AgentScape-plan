# Verified Baseline — 2026-08-27

本文件记录迁移开始前已经真实成立的能力。重构不得用“架构更漂亮”作为回退这些能力的理由。

# 1. Pinned Workspace Heads

```text
AgentScape                         ad17111
AgentScape-client                  3a4a2d2
modal-gen-client                   4e93fc1
modal-2D-client                    72ff9bb
modal-2D                           e237b30
modal-3D-client                    d92fabe
modal-3D                           a814f1d
kaggle-inference-hub               334de7c
modal-build                        7aca4e8
EmbodiedGen upstream               f012419
modal-lab                          e266aba
AgentScape-plan before rewrite     689cba4
```

这些 commit 是本次架构计划重建时的 workspace 事实，不代表所有仓库都已经完成目标架构。

# 2. Real Apple E2E

已真实验证：

```text
prompt
→ real modal-2D PNG
→ real background preprocess
→ real FastSAM3D++ / modal-3D GLB
→ Artifact import / Asset Compiler
→ WorldPipeline
→ table placement
→ support.on == true
→ Agent autonomous navigation
→ pickup
→ heldBy == agent_01
→ behavior verification == true
```

最近验收事实：

```text
real E2E process exit = 0
relation admission = ready
support.on = true
agent traveled ≈ 0.99m
pickup status = held
verifyBehaviorCommand.verified = true
```

# 3. Test Baseline

已记录的最近通过证据：

```text
AgentScape targeted architecture/E2E regression: 28/28 PASS
modal-3D-client full pytest after latest-base replay: 169/169 PASS
```

历史 root full test 曾存在 3 个 nested React frontend collection failure（root node_modules 缺 nested frontend React dependencies）；不能把它写成“full root suite 已全绿”。

# 4. Durable Execution Facts

已经验证并必须保留：

- expensive remote submit 使用稳定 request identity / idempotency。
- modal-3D-client 能表达 remote-created/uncertain/recovery，而不是超时后盲目重复提交。
- modal-3D capability discovery 使用 last-known-good/cache 以避免 Modal 冷启动误判 Provider unavailable。
- persisted preprocess model 在进程重启后可通过本地完整性校验恢复 `ready/verified`。
- Gateway pairing 使用 scoped session/origin，privileged secret 不应暴露给 Browser。

# 5. Asset / World Facts

- Provider Artifact 不直接决定 Asset ready。
- Asset Compiler 当前已有 semantic/physics/collider/quality/admission 机制。
- WorldPipeline 已有 resolve/admission/layout/behavior/physics/instantiate/relation/validate/repair/finalize 分阶段逻辑。
- `ON table.top` 已通过 SpatialSystem `supportStatus(...).on == true` 做硬验证。
- PICKUP 通过实际 Navigation/Locomotion/Physics/InteractionSystem 执行，不是 mock 业务路径。

# 6. Embodied Evidence Facts

旧计划中已有真实 Embodied evidence/bundle 工作；迁移必须保留以下原则：

```text
Provider evidence
≠ semantic truth
≠ simulation validation
≠ AgentScape runtime verification
```

Segmentation/raw grasp/semantic/SAPIEN/AgentScape evidence level 不得合并。

# 7. Baseline Rule

每个迁移切片必须回答：

```text
本切片改变了什么 Ownership？
哪些 baseline tests/smokes 证明没有回退？
新的独立验证入口是什么？
Legacy 何时可以删？
```


# 8. Architecture Migration Evidence — R2 / modal-2D

2026-08-27 第一组正式迁移已完成：

```text
modal-2D         e237b30  feat: stabilize modal 2d artifact contract
modal-2D-client  72ff9bb  feat: stream modal 2d artifacts from volume
AgentScape       ad17111  chore: sync modal 2d artifact migration
```

验证证据：

```text
modal-2D ruff                      PASS
modal-2D pytest                    18/18 PASS
modal-2D-client ruff               PASS
modal-2D-client pytest             36/36 PASS
modal-gen-client full pytest       32 PASS / 2 SKIP
AgentScape targeted regression     29/29 PASS
real modal-2D deploy               PASS
real capability metadata           PASS
real PNG bytes                     808259
real PNG dimensions                1024x1024
real Artifact digest               MATCH
real sidecar Volume-first fetch    PASS
legacy read_artifact fallback used false
```

迁移后的稳定边界：

```text
modal-2D
  GPU → PNG → named Volume + ArtifactDescriptor
                    │
                    ▼
modal-2D-client
  Volume-first stream → integrity verify → content-addressed cache
                    │
                    ▼
Connector / caller
```

`remote_path` / Modal Volume 仍是 Provider-private transport；AgentScape 不感知该位置。


# 9. Provider Verification Experiments — 040 / 041

2026-08-27 已完成真实 Provider baseline：

```text
040-modal-2d-provider   PASS
041-modal-3d-provider   PASS
```

## modal-2D

```text
SANA-Sprint 0.6B / seed 42   PASS   834149 bytes
SANA-Sprint 0.6B / seed 73   PASS   818196 bytes
SANA-Sprint 1.6B / seed 42   PASS  1026180 bytes
SANA-Sprint 1.6B / seed 73   PASS   675018 bytes
```

所有候选：real GPU、1024×1024 PNG、Volume read、bytes/SHA-256/producer 一致；同模型不同 seed digest 不同。

## modal-3D preprocessing baseline

```text
engine              birefnet-general-lite
execution           cloud
foreground ratio    0.2843132019042969
component count     1
canonical            1024×1024 RGBA
alpha extrema        0–255
```

## modal-3D model matrix

```text
FastSAM3D++           PASS   7,515,508 bytes   67.520s
Hermite-TRELLIS2++    PASS  36,759,736 bytes  373.589s
Hunyuan2.1++          PASS  43,326,464 bytes  725.648s
Pixal3D               PASS  35,423,056 bytes  366.035s
```

四个模型均满足：

```text
same model + same input + same options
→ duplicate gateway submit
→ same callId

GLB magic = glTF
GLB version = 2
declared bytes = actual bytes
SHA-256 = descriptor SHA-256
```

这些结果是后续 `modal-3D InputConditioner` public-contract 迁移的 baseline；迁移后必须重跑同一实验矩阵并保持 parity。


# 10. AgentScape-agent Verified One-shot Vertical Slice

2026-08-28，独立 `AgentScape-agent` 已完成从文本到真实 Runtime World 的 one-shot Vertical Slice：

```text
AgentScape-agent  43874d4  feat: rank image candidates before one-shot world generation
```

代码形态继续遵循 Single-file First：

```text
src/agent.js              high-level Agent tool loop
src/source_3d_asset.js    Text → candidates → VLM select → 3D → Asset Vertical Slice
src/run_text_to_world.js  source_3d_asset → World → Runtime verify one-shot composition
src/runs.js               仅拥有独立 checkpoint / failure-recovery 生命周期
```

最终本地 Gate：

```text
node:test                         34/34 PASS
node --check                      PASS
source_3d_asset replay            5/5 PASS
Agent trajectory replay           5/5 PASS
git diff --check                  PASS
tracked secret scan               PASS (no credential in tracked files)
```

真实多候选旗舰链：

```text
Text
  ↓
4 × modal-2D-client / SANA-Sprint 1.6B
  ↓
OpenAI-compatible Vision Ranker
  ↓
stepfun-ai/step-3.7-flash
  ↓
selected candidate
  ↓
modal-3D-client / FastSAM3D++
  ↓
verified GLB
  ↓
AgentScape ArtifactRegistry + SHA-256 verify
  ↓
AssetCompiler / Asset admission
  ↓
canonical WorldPipeline
  ↓
ON table placement
  ↓
WorldRuntime support verification
```

最终 rebase 后真实复验：

```text
status                            completed
stage                             verified
elapsed                           99.901 s
candidateCount                    4
2D model                          sana-sprint-1.6b
2D seeds                          42 / 73 / 104 / 135
2D jobs                           4/4 succeeded
VLM                               stepfun-ai/step-3.7-flash
VLM selected seed                 42
3D model                          fastsam3d-plus-plus
3D artifact bytes                 7,853,800
3D GLB SHA-256                    120a9658ffad6a6c3d7232b9a717ce9737279334d87ce04b245c8e5085b0422e
Asset admission                   provisional
Asset reason                      BUDGET_RENDER_VERTICES
World admission                   provisional
World reason                      ASSET_PROVISIONAL
relation admission                ready
object ON table                    verified
Runtime object                     present
```

`provisional` 不是失败：当前 GLB 超过既有 render-vertex budget，因此 Asset 保持 provisional，World 继承该 admission；但空间关系与 Runtime 支撑真值均已验证，one-shot 只有在 `world.verified=true` 时才返回 `completed`。

另外真实修正了 2D Sidecar job identity 语义：`modal-2D-client.job_id` 是唯一 Job ID，不是跨 Run 幂等 request key。`createModal2DAdapter()` 现在为每次 `generateImages()` 创建独立 run scope；同一 Run 内候选保持可追踪，不同 Run 不再复用历史中断 Job。该修复与 `AbortSignal → Sidecar DELETE → remote cancellation` 的取消传播能力同时通过 34/34 tests。

当前仍未完成、必须继续明确标记的内容只有：**跨进程 Tool resume / 自动恢复**。不为了 roadmap checkbox 额外抽 `build_world` service；当前 `run_text_to_world.js` 已形成深而窄的 one-shot composition，尚无独立 State Owner、failure lifecycle、deployment 或性能压力支持进一步 Extract。


# 11. AgentScape Asset / World Modular Boundary — 2026-08-28

已完成五个独立、逐步验证并推送的迁移切片：

```text
7a7adea  refactor: establish asset and world module boundaries
bc3be81  refactor: move authoring out of world runtime
7bafc7c  refactor: move provider generation behind asset port
9fb7fec  refactor: isolate world execution behind asset refs
0a41a93  refactor: move asset state ownership into asset module
9369e12  feat: stabilize asset publication api
a0b522a  refactor: retire asset library generation compatibility
86a2232  refactor: remove redundant asset library facade
```

当前结构证据：

```text
Asset state owner
  src/assets/createAssetModule.js
  ├─ AssetManager
  ├─ CompiledAssetStore
  └─ AssetCatalog

Asset → World contract
  AssetRef { assetId }

WorldRuntime
  requires injected assetModule
  imports no Asset implementation
  imports no Connector / Provider / Generation authoring

World Compilation v2
  assetRequests = authoring compatibility only
  entities      = execution projection with AssetRef
```

独立 Gate：

```text
Asset tests                    107/107 PASS
World tests                    165/165 PASS
AgentScape root tests          746/746 PASS
production build               PASS
repository architecture        PASS (11 pinned submodules)
domain architecture            PASS
asset validation               PASS
```

Asset experiment：

```text
real FastSAM3D++ GLB           7,515,508 bytes
Artifact → Compiler → Admission → Catalog  PASS
Asset                          experiment_red_apple
Admission                      provisional
Reason                         BUDGET_RENDER_VERTICES
Searchable                     true
```

World experiment：

```text
Provider generation            NOT USED
existing AssetRefs             table + cup
relation                       cup_01 ON table_01
World admission                ready
execution entities             AssetRef only
query/generate/provider leak   none
```

结论：独立测试能力已经在同一 Repository 内成立；当前没有证据要求立即物理拆分 `AgentScape-Asset` / `AgentScape-World`。继续按 Extract by Pressure 观察 state lifecycle、release cadence、dependency/failure isolation 与 change coupling。


## 11.1 Stable Asset Publication API — 2026-08-28

`9369e12 feat: stabilize asset publication api` 完成 Caller → Asset 的稳定发布接缝：

```text
Verified Artifact
      │
      ▼
assetModule.publishAsset({ artifactId, assetId, label })
      │
      ├─ integrity / availability gate
      ├─ lease + idempotency
      ├─ AssetCompiler
      ├─ Asset admission
      ├─ Asset registration
      └─ AssetRef { assetId }
```

Ownership：

```text
createAssetModule()
  ├─ ArtifactRegistry
  ├─ ArtifactByteStore
  ├─ CompiledAssetStore
  ├─ AssetManager
  ├─ AssetCatalog
  └─ publishAsset()
```

`GenerationOrchestrator` 不再拥有/构造 `AssetManager`、`AssetCompiler`、publication pipeline、Artifact Registry 或 Artifact ByteStore；它仅 import Provider Artifact 并调用注入的 `publishAsset()`。

验证：

```text
AgentScape root tests          746/746 PASS
production build               PASS
repository/domain architecture PASS
asset validation               PASS
real modal-lab SF3D GLB        803,592 bytes
public publishAsset experiment PASS
returned AssetRef              PASS
idempotent reuse               PASS
provenance conflict fail-close PASS
compiler rejection no-register PASS
```


## 11.2 Asset Read Boundary — 2026-08-28

`a0b522a` 先完成 generation 退出 AssetLibrary；`86a2232` 随后删除冗余 AssetLibrary facade，使 AssetCatalog 成为唯一 Asset read API：

```text
AssetCatalog
  ├─ has/get/list/search/summary
  └─ resolveExisting

LegacyAuthoringShell
  ├─ canGenerateAsset
  ├─ generateAsset
  ├─ resolveAssetRequest
  └─ AssetGenerationPort
```

禁止回退：

```text
AssetLibrary                   REMOVED
runtime.assetLibrary alias     REMOVED
UI Asset reads                 AssetCatalog only
Authoring generation           LegacyAuthoringShell only
```

Architecture validator 现在验证 AssetCatalog 为唯一 Asset read facade；生产代码中 `AssetLibrary/assetLibrary` 引用已归零。

验证：

```text
targeted migration tests       53/53 PASS
AgentScape root tests          746/746 PASS
production build               PASS
domain architecture            PASS
Asset/World independent gates  PASS
```


## 11.3 Single Asset Read API — 2026-08-28

`86a2232 refactor: remove redundant asset library facade` 删除最后一个浅包装模块：

```text
BEFORE
AssetLibrary → AssetCatalog

AFTER
AssetCatalog
  ├─ list
  ├─ search
  ├─ get
  └─ resolveExisting
```

结果：

```text
AssetLibrary refs             0
runtime.assetLibrary alias    removed
UI reads                      AssetCatalog directly
LegacyAuthoring               AssetCatalog + generation port
AgentScape root tests         746/746 PASS
production build              PASS
architecture validation       PASS
Asset experiment              PASS
World experiment              PASS
```

测试数量从 748 降至 746 仅因为删除 `tests/asset-library.test.js` 中两个已无意义的 facade tests，不是功能回归。


## 11.4 End-to-End Cancellation Ownership — 2026-08-28

取消链已经从 Caller 贯穿到真实 Sidecar durable Job：

```text
AbortSignal
   ↓
Agent Run
   ↓
high-level Tool
   ↓
source_3d_asset
   ↓
2D / VLM / 3D adapter
   ↓
DELETE deterministic jobId
   ↓
Sidecar cancel_requested
   ↓
remote cancellation
   ↓
cancelled
```

关键修复提交：

```text
modal-2D-client
625c72c  fix: honor cancellation during image submission

modal-3D-client
32ab2f1  fix: preserve 3d cancellation intent while polling
abe29a1  fix: honor cancellation during remote submission
5f8510f  fix: keep cancellation pending until remote bind settles
cf6526b  fix: keep sidecar responsive during 3d submission

AgentScape-agent
0c52996  feat: propagate cancellation through agent workflows
f4add25  docs: record verified cancellation propagation

AgentScape
b5bf79f  chore: sync cancellation sidecar fixes
```

验证：

```text
modal-2D-client ruff             PASS
modal-2D-client pytest           38/38 PASS
modal-3D-client ruff             PASS
modal-3D-client pytest           25/25 PASS
AgentScape-agent node:test       34/34 PASS
source_3d_asset replay           5/5 PASS
Agent trajectory replay          5/5 PASS
AgentScape architecture validate PASS (11 pinned submodules)
```

真实 2D 取消：

```text
jobId       agent2d_9f0b6f515b54d6a0b9775a27
final       cancelled
errorCode   remote.cancelled
retryable   false
```

真实 3D 取消：

```text
jobId       agent3d_bf6c61e8ca725a8bd7b1e7b0
appeared    running
final       cancelled
errorCode   remote.cancelled
retryable   false
```

3D Sidecar 额外修复了 async HTTP event-loop 被同步 Modal submit 阻塞的问题：`POST /v1/jobs` 现在把同步 submit 放入 FastAPI threadpool，因此同一 Sidecar 在提交期间仍能处理 GET/DELETE cancellation request。Sidecar 在 remote call 尚未绑定时保留 `cancel_requested`，remote id 返回后重新读取最新 durable state，并在需要时立即取消刚绑定的 remote call，避免后台 GPU Job 泄漏。
