# Verified Baseline — 2026-08-27

本文件记录迁移开始前已经真实成立的能力。重构不得用“架构更漂亮”作为回退这些能力的理由。

# 1. Pinned Workspace Heads

```text
AgentScape                         5a381d5
AgentScape-client                  3a4a2d2
modal-gen-client                   4e93fc1
modal-2D-client                    dd21b37
modal-2D                           8a8e6ed
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
