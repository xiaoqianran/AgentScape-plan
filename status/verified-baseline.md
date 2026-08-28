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

> **2026-08-28 repository convergence:** `AgentScape-client@3a4a2d2` 的完整 18-commit 历史已并入 `AgentScape/sdk/python`，独立 GitHub repository 已删除。上面的 pinned head 仅保留为迁移前历史事实。

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
AgentScape-agent  f8400a3  feat: generate image candidates through provider batches
AgentScape-agent  638a553  feat: resume interrupted tools across processes
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
node:test                         35/35 PASS
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
2D batch jobs                     1/1 succeeded
2D artifacts                      4/4 verified
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

2D candidate 路径已进一步收敛为一个 Provider batch Job：同一 prompt 的 `seeds=[42,73,104,135]` 只创建一个 `modal-2D-client` Job / 一个 Modal `FunctionCall` / 一个 `SanaSprintWorker`，Provider 返回 `artifacts[]`。同一个 `executionId` 跨进程恢复时 rebind 同一个 batch Job，不重复创建 GPU execution。`AbortSignal → Sidecar DELETE → remote cancellation` 与 batch rebind 同时通过 35/35 tests。

跨进程 Tool resume / 自动恢复已经通过“两进程 crash → load checkpoint → same executionId → recovered Tool → completed”的真实实验。控制面恢复开销为几十毫秒量级；不为了 roadmap checkbox 额外抽 `build_world` service，当前 `run_text_to_world.js` 继续保持深而窄的 one-shot composition。


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
ef2830d  refactor: move generation orchestration into authoring
4dab4f7  refactor: move generation job center into authoring
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


## 11.4 Generation Orchestration Leaves Core — 2026-08-28

`ef2830d refactor: move generation orchestration into authoring` 将 generation orchestration 的物理 ownership 从 Core 路径迁出：

```text
BEFORE
src/generation/GenerationOrchestrator.js

AFTER
src/authoring/GenerationOrchestrator.js
```

当前职责：

```text
src/authoring/
  ├─ LegacyAuthoringShell
  ├─ GenerationOrchestrator
  └─ GenerationJobCenter

src/generation/
  └─ <no production modules>
```

验证：

```text
targeted orchestrator tests   12/12 PASS
AgentScape root tests         746/746 PASS
production build              PASS
architecture validation       PASS
Asset experiment              PASS
World experiment              PASS
stale old orchestrator path   0
```

说明：这完成的是 **GenerationOrchestrator 从 Asset/World Core 移出**，不是 legacy authoring 功能的最终删除；后者等待外部 Caller/Agent parity 后再 retire。


## 11.5 Generation Job Center Leaves Core — 2026-08-28

`4dab4f7 refactor: move generation job center into authoring` 完成 generation UI/control-plane ownership 的物理收口：

```text
BEFORE
src/generation/GenerationJobCenter.js

AFTER
src/authoring/GenerationJobCenter.js
```

最终目录：

```text
src/authoring/
  ├─ LegacyAuthoringShell.js
  ├─ GenerationOrchestrator.js
  └─ GenerationJobCenter.js

src/generation/
  └─ no production modules
```

Architecture validator 现在 fail-closed：`src/generation/` 下重新出现生产 `.js` 文件会失败。

验证：

```text
GenerationJobCenter targeted   8/8 PASS
AgentScape root tests          746/746 PASS
production build               PASS
architecture validation        PASS
Asset experiment               PASS
World experiment               PASS
src/generation production files 0
```


## 2D Candidate Batch Performance — 2026-08-28

已完成 `modal-2D` / `modal-2D-client` / `AgentScape-agent` 的 candidate batch 迁移：

```text
modal-2D         1299287  feat: batch image candidates on one warm worker
modal-2D-client  6ca829c  feat: mirror provider candidate batches as one job
AgentScape-agent f8400a3  feat: generate image candidates through provider batches
AgentScape       ab527f9  perf: batch modal image candidates on one worker
```

最终 Gate：

```text
modal-2D pytest                  20/20 PASS
modal-2D ruff                    PASS
modal-2D-client pytest           41/41 PASS
modal-2D-client ruff             PASS
AgentScape-agent node:test       35/35 PASS
Agent source replay               5/5 PASS
Agent trajectory replay           5/5 PASS
real modal-2D deploy             PASS
real cold→warm batch             PASS
```

最终结构：

```text
prompt + seeds[42,73,104,135]
            │
            ▼
ONE modal-2D-client Job
            │
            ▼
ONE submit_batch FunctionCall
            │
            ▼
ONE SanaSprintWorker / L40S
  ├─ seed 42
  ├─ seed 73
  ├─ seed 104
  └─ seed 135
            │
            ▼
4 verified PNG artifacts
```

连续 cold → warm 实测：

```text
pre-batch 4 independent jobs     ~54.2 s
cold one-batch job                43.362 s
warm one-batch job                 9.075 s
warm Provider batch compute        6.782 s
```

Warm worker 内单图 inference：

```text
seed 42     1.352 s
seed 73     1.353 s
seed 104    1.240 s
seed 135    2.428 s
```

Warm 返回 `worker_reused=true / worker_load_ms=null`；cold 独立记录 `worker_load_ms=16.909s`。Provider `scaledown_window=300s`，generation hot path 不再执行同步 `prefetch.remote()`；`prefetch` 保留为显式模型准备 capability。

# 12. modal-3D Provider Input Conditioning — 2026-08-28

`modal-3D@487b661 feat: condition source images inside modal 3d` 已将
InputConditioner 下沉到 Provider，公开输入契约不再强制 canonical RGBA。

**这是 CODE DONE，不是 verified。** 记录于此是为了防止回退，不是为了宣称
`041` parity 已达成。

## 已验证（代码层 Gate）

```text
modal-3D unittest discover -s tests     87/87 PASS   （与 CI 一致）
modal-3D pytest tests/                  87/87 PASS
modal-3D python -m compileall           PASS
```

CI 使用 `unittest discover -s tests`（见 `.github/workflows/ci.yml`），不是 pytest。

## 公开契约变更

```text
BEFORE（041 baseline）
public input = canonical 1024×1024 RGBA，alpha_required
Caller 负责 rembg / crop / canonicalize

AFTER
public input = image/png | image/jpeg | image/webp
               maxBytes 20 MiB
               alpha optional
               conditioning = provider
               pathPrefix = source-inputs/
Provider 负责 decode → alpha 判定 → 必要时 rembg → letterbox → canonical
```

## Conditioning 策略

```text
已有 meaningful alpha
  → strategy = preserve-alpha
  → 原 alpha 直接保留

opaque source
  → RemBgWorker 预测 mask
  → refine_mask（binary_fill_holes + binary_closing，BiRefNet tail）
  → strategy = birefnet
  → 记录 engine / mask_elapsed_ms

两条分支后统一
  → foreground bbox → letterbox → canonical 1024×1024 RGBA
```

## 041 Parity 保护机制（关键，禁止回退）

```text
client-inputs/<sha>.png
  → _legacy_canonical() pass-through
  → 字节原样保留
  → source_sha256 == canonical_sha256
  → strategy = legacy-canonical-pass-through
  → 041 四模型矩阵行为与迁移前完全一致

source-inputs/<sha>.<ext>
  → condition_image() 新路径
```

`rembg_gateway.condition()` 的分支判定在 `rembg_gateway.py:214`；
注释已明确写出保留 legacy 字节是为了让 `041` 继续作为 strict parity gate。
删除这条 pass-through 前必须先取得 conditioning parity 证据。

## Worker 契约未变

```text
capabilities.py:71
  raise ValueError("worker input contract must be canonical 1024x1024 RGBA PNG")
```

Conditioning 发生在 Gateway/Provider 边界内，四个 Worker
（FastSAM3D++ / Hermite-TRELLIS2++ / Hunyuan2.1++ / Pixal3D）
仍只认 canonical，不感知 conditioning 存在。

## 尚未验证 — 重跑 041 前不得标记 verified

```text
[ ] source-inputs/ 路径四模型矩阵
[ ] conditioning 后 GLB digest 与 legacy canonical 路径一致或差异已解释
[ ] strategy 分布（preserve-alpha vs birefnet）真实采样
[ ] GLB magic / version / declared bytes / SHA 全匹配
```

## 已知非阻塞问题

```text
archive/sam3_1/test_sam3_materialize.py
  导入已删除的 modal_3d.sam3_1
  pytest 全仓收集即失败（1 error）
  487b661 之前已存在，不属于本次 Gate
  CI 只跑 tests/，不受影响

ruff check modal_3d/ tests/
  7 errors（未使用变量等）
  ruff 不在 CI 与 dev dependency group 中，不构成 Gate
```
