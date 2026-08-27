# Integration Ledger

本文件记录**当前真实箭头**与目标判定。每次跨仓迁移必须更新本表。

当前代码审计锚点：

```text
AgentScape/src/runtime/WorldRuntime.js
AgentScape/src/generation/GenerationOrchestrator.js
AgentScape/src/providers/ProviderRegistry.js
AgentScape/src/assets/library/AssetLibrary.js
AgentScape/src/adapters/EmbodiedGenAdapter.js
modal-gen-client/modal_gen/jobs.py
modal-gen-client/modal_gen/storage.py
modal-3D-client/agent/projects.py
modal-3D-client/agent/jobs.py
modal-3D-client/agent/generation_store.py
modal-3D-client/agent/connector/*
kaggle-inference-hub/hub/app.py
kaggle-inference-hub/hub/state.py
modal-build/runtime/embodiedgen_v2_l40s.py
```

状态：`KEEP` 保留；`MOVE` 移动 Owner；`SIMPLIFY` 收缩；`REMOVE` 最终删除；`ADD` 新增目标边界；`VERIFY` 当前没有足够证据，不假装存在。

## 1. AgentScape 内部到 Provider 链

| From → To | 当前用途 | 当前 Owner/问题 | Target | Verdict |
|---|---|---|---|---|
| `WorldRuntime → ConnectorClient` | Runtime 初始化生成连接 | World Domain 知道 transport/Provider | Provider 创建移出 Runtime；Runtime 只消费 Asset | **REMOVE** |
| `WorldRuntime → GenerationOrchestrator` | Runtime 暴露生成能力 | Runtime 与 generation lifecycle 耦合 | 移到 Asset Sourcing/Application composition | **MOVE** |
| `AssetLibrary → GenerationOrchestrator` | miss 后生成资产 | Library 同时 search/resolve/generate | AssetLibrary 只管理 existing assets；生成由 Asset Sourcing | **REMOVE** |
| `GenerationOrchestrator → ProviderRegistry` | discovery/select/execute/composition | Orchestrator 同时拥有太多 lifecycle | Catalog 只 resolve；执行/组合留在 Asset Sourcing 的小状态机 | **SIMPLIFY** |
| `GenerationOrchestrator → Asset Compiler` | 生成后导入/compile | execution 与 domain compile 混合 | Execution 产 Artifact；AssetSourcing 显式 verify → compile | **MOVE** |
| `AgentScape → EmbodiedGenAdapter` | provider payload/evidence 适配 | legacy adapter 可直接构造 provisional manifest | bundle/artifact/evidence → verifier → existing AssetCompiler | **SIMPLIFY** |

## 2. AgentScape / Client 到 Local Gateway

| From → To | 当前用途 | Target | Verdict |
|---|---|---|---|
| `AgentScape ConnectorClient → modal-gen-client` | capability/jobs/artifacts transport | 保留为一个可选 Transport Adapter，不是唯一 Provider 路径 | **KEEP + SIMPLIFY** |
| `AgentScape-client → modal-gen-client` | reference client / local gateway 调用 | 保留 optional gateway transport | **KEEP** |
| Browser/WebView → `modal-gen-client` | 本机 privileged boundary | pairing/origin/scope/secret isolation | **KEEP** |

## 3. Local Gateway 到 Provider Sidecar

| From → To | 当前用途 | Target | Verdict |
|---|---|---|---|
| `modal-gen-client → modal-2D-client` | 2D capability/jobs/artifact adapter | 机械 transport adapter；不含 image workflow 策略 | **KEEP + SHRINK** |
| `modal-gen-client → modal-3D-client` | 3D capability/jobs/artifact adapter | 机械 transport adapter；只调用 Application API | **KEEP + SHRINK** |
| Gateway storage → provider job/artifact refs | 本地 job/artifact projection | 明确标注 projection，不成为 canonical execution/artifact truth | **SIMPLIFY** |

## 4. Sidecar 到 Provider Runtime

| From → To | 当前用途 | Target | Verdict |
|---|---|---|---|
| `modal-2D-client → modal-2D` | Modal remote execution + Volume-first Artifact fetch | local durable mirror → provider execution → verified PNG cache | **KEEP / MIGRATED 2026-08-27** |
| `modal-3D-client → modal-3D` | Modal remote 3D execution | Generation Saga → provider execution → GLB | **KEEP** |
| `modal-3D-client connector/* → internal stores` | Connector compatibility | Connector 只能调用 Application Boundary，不直接操作 Domain Store | **MOVE** |

## 5. Embodied / Build / Research

| From → To | 当前用途 | Target | Verdict |
|---|---|---|---|
| `modal-build → EmbodiedGen` | pinned source / patch / build / runtime | Build Plane 唯一 compatibility owner | **KEEP** |
| AgentScape → Embodied provider bundle/evidence | consume 3D/affordance evidence | Artifact/evidence bridge → existing verifier/compiler；不构造 verified manifest | **SIMPLIFY** |
| AgentScape → `modal-lab` | 无 canonical production dependency | 永久保持无 runtime dependency | **KEEP ABSENT** |
| `modal-lab → production repo` | 手工实验成果迁移 | 通过 reproducible/evidence/promotion gate 重写进入 production | **ADD PROCESS** |

## 6. Kaggle

当前 `kaggle-inference-hub` 自身已明确拥有 task/claim/heartbeat/upload/history 等 queue/worker 协议；**当前 AgentScape 主链是否存在 canonical direct adapter 不作为既定事实**。

目标新增边界：

```text
AgentScape / Reference Client
          │ Consumer Contract
          ▼
kaggle-inference-hub Consumer API
          │
          ▼
Queue / Lease / Worker Protocol
```

在真正启用 AgentScape runtime adapter 前必须先完成 Consumer API 与 Worker API 分离。

**Verdict：VERIFY CURRENT / ADD TARGET AFTER KAGGLE MIGRATION。**

## 7. Integration Rule

任何新箭头必须写清：

```text
Purpose
Contract Owner
Transport Owner
State Owner
Credential Owner
Retry Owner
Artifact Owner
Failure mapping
```

如果一条箭头需要共享对方私有 Store/ORM/内部 class 才能工作，默认判定为架构违规。
