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


# 10. AgentScape-agent Initial Vertical Slice

2026-08-27 已建立独立 `AgentScape-agent` 仓库并完成第一版 Experiment-oriented Modular Monolith：

```text
AgentScape-agent  e8af6ab  feat: preflight sidecar capabilities
```

代码形态：

```text
src/agent.js            high-level Agent tool loop
src/source_3d_asset.js  single-file Vertical Slice
```

验证：

```text
node:test                         25/25 PASS
node --check                      PASS
real modal-2D-client health       PASS
real modal-2D candidate run       4/4 PASS
model                             sana-sprint-1.6b
seeds                             42 / 73 / 104 / 135
elapsed                           63672 ms
candidate SHA-256                 4 distinct digests
VLM ranking                       SKIPPED (credentials unavailable)

source_3d_asset replay            5/5 PASS
checkpoint atomic JSON            PASS
checkpoint mode                   0600
checkpoint failure                fail-closed
binary Tool observation           rejected before persistence
real modal-3D-client adapter      PASS
3D model                           fastsam3d-plus-plus
3D jobId                           agent3d_193038dd45916f91ea1b4437
3D artifactId                      art_adf3b2520c19532daad5197a984e2
3D artifact bytes                  7,525,252
3D GLB SHA-256                     543694494b4482d053d1eaae47e84cdb08f9170287ad319f6be66d40fa0fb667
3D conditioning                    birefnet / birefnet-general-lite
3D foreground ratio                0.2843132019042969
3D source SHA-256                  MATCH
3D elapsed                         220643 ms
real sidecar capability preflight PASS
2D capability                      sana-sprint-1.6b / available
3D capability                      fastsam3d-plus-plus / recommended / enabled
unknown model/profile              fail-closed before Job submit
capability fetch                   lazy + cached per adapter
```

已验证的边界：

```text
Agent
  → high-level tool only
  → source_3d_asset
      → deterministic pure state transitions
      → side effects in imperative shell
      → real modal-2D sidecar adapter
      → OpenAI-compatible VLM contract adapter
```

未验证的内容必须继续标记为未完成：真实 VLM ranking、跨进程 Tool resume。Asset publication 已由 `9369e12` 独立验证；World placement 已由 provider-free World experiment 独立验证。


# 11. AgentScape Asset / World Modular Boundary — 2026-08-28

已完成五个独立、逐步验证并推送的迁移切片：

```text
7a7adea  refactor: establish asset and world module boundaries
bc3be81  refactor: move authoring out of world runtime
7bafc7c  refactor: move provider generation behind asset port
9fb7fec  refactor: isolate world execution behind asset refs
0a41a93  refactor: move asset state ownership into asset module
9369e12  feat: stabilize asset publication api
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
AgentScape root tests          747/747 PASS
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
AgentScape root tests          747/747 PASS
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
