# Integration Ledger

本文件记录**当前真实箭头**和**目标箭头**。Architecture Card 描述 Target；Ledger 负责防止我们把 Target 当成 Reality。

状态：`KEEP` 保留；`MOVE` 改 Owner；`SIMPLIFY` 收缩；`REMOVE` 删除；`ADD` 新增；`MIGRATED` 已完成并有证据；`VERIFY` 尚未确认。

## 1. Caller Layer

| Arrow | Current | Target | Verdict |
|---|---|---|---|
| `User Text → AgentScape internal ToolCallingAgent` | Agent 仍在 AgentScape Core 内 | `User Text → AgentScape-agent` 独立仓 | **MOVE / ADD REPO** |
| `AgentScape-agent → modal-2D-client` | 不存在 | Agent image-generation Tool Adapter | **ADD** |
| `AgentScape-agent → modal-3D-client` | 不存在 | Agent 3D-generation Tool Adapter | **ADD** |
| `AgentScape-agent → AgentScape` | 不存在 | search/publish Asset + build/observe World | **ADD** |
| `Human → old modal-3D-client UI` | 存在；仓库同时拥有 Project/Preprocess/3D execution | `Human → modal-inference-hub` | **RENAME + PURIFY** |
| `modal-inference-hub → modal-2D-client` | 尚未形成稳定边界 | image candidate workflow | **ADD** |
| `modal-inference-hub → modal-3D-client` | 旧 Hub 仍直接执行 3D | 改为纯 Sidecar 调用 | **MOVE** |
| `modal-inference-hub → AgentScape` | 非核心稳定路径 | optional publish Asset / inspect World | **ADD OPTIONAL** |

## 2. AgentScape Current Coupling

当前代码审计锚点：

```text
AgentScape/src/runtime/WorldRuntime.js
AgentScape/src/generation/GenerationOrchestrator.js
AgentScape/src/providers/ProviderRegistry.js
AgentScape/src/assets/library/AssetLibrary.js
AgentScape/src/adapters/EmbodiedGenAdapter.js
```

| Arrow | Current Problem | Target | Verdict |
|---|---|---|---|
| `WorldRuntime → ConnectorClient` | World Domain 知道 transport/provider | WorldRuntime 只消费 compiled Asset/World | **REMOVE** |
| `WorldRuntime → GenerationOrchestrator` | Runtime 与 generation lifecycle 耦合 | generation 移到 Caller (`AgentScape-agent`/Hub) | **REMOVE** |
| `AssetLibrary → GenerationOrchestrator` | Repository 同时 search + generate | AssetLibrary 只 read/search/get | **MIGRATED 2026-08-28 (`a0b522a`)** |
| `GenerationOrchestrator → ProviderRegistry` | discovery/execute/composition 混合 | Agent/Hub Tool Adapter 调 Sidecar | **RETIRE** |
| `GenerationOrchestrator → AssetCompiler` | provider execution 和 domain compile 混合 | `assetModule.publishAsset(Artifact)` | **MIGRATED 2026-08-28 (`9369e12`)** |
| `AgentScape internal Agent → provider jobs` | Agent sees low-level execution surface | 独立 Agent 选择 Skill；Skill 隐藏 poll/download | **MOVE** |

## 3. modal-2D Path

| Arrow | Current | Target | Verdict |
|---|---|---|---|
| `modal-2D-client → modal-2D` | durable mirror + Volume-first Artifact fetch | 保持 | **MIGRATED 2026-08-27** |
| `modal-gen-client → modal-2D-client` | Connector adapter 已存在 | optional security transport only | **KEEP + SHRINK** |
| `AgentScape-agent → modal-2D-client` | 不存在 | prompt → candidate image jobs | **ADD** |
| `modal-inference-hub → modal-2D-client` | 不存在稳定调用 | human image candidate generation | **ADD** |

真实 Gate：`040-modal-2d-provider`。

## 4. modal-3D Path

当前现实：

```text
old modal-3D-client (renamed remote → modal-inference-hub)
  owns Project + rembg/preprocess + Modal session + generation Job + GLB artifact
                    │
                    ▼
                 modal-3D
```

目标：

```text
modal-inference-hub               AgentScape-agent
        │                               │
        └──────────────┬────────────────┘
                       ▼
               modal-3D-client
        transport / durable execution
                       │
                       ▼
                   modal-3D
                 InputConditioner
                       │
                       ▼
                   3D Worker
```

| Arrow | Current | Target | Verdict |
|---|---|---|---|
| `old Hub generation code → modal-3D` | 直接 Modal execution | Hub → new modal-3D-client | **MOVE** |
| `old Hub preprocess → canonical RGBA` | Project-level rembg/component/canonical | Human semantic selection 留 Hub；model-required conditioning 下沉 Provider | **SPLIT OWNERSHIP** |
| `new modal-3D-client → modal-3D` | 新仓已建立；初版要求 canonical RGBA | Sidecar 上传 image + optional mask，不拥有 rembg | **ADD / ADJUST CONTRACT** |
| `modal-3D Provider API → canonical RGBA` | 当前公开 contract | public image/mask → internal canonical | **MOVE CONDITIONING INTO PROVIDER** |
| `modal-gen-client → old 3D client` | 现有 adapter 指向旧应用 | 指向 new modal-3D-client | **MOVE** |

真实 Gate：`041-modal-3d-provider`，先记录当前 canonical baseline，再迁 InputConditioner。

## 5. Candidate Selection

| Responsibility | Owner |
|---|---|
| Human manual selection/mask edit | `modal-inference-hub` |
| Agent/VLM candidate ranking | `AgentScape-agent` |
| Provider automatic model conditioning | `modal-3D` |
| Sidecar transport validation | `modal-3D-client` |

禁止把“用户想选哪个物体”和“模型需要怎样 crop/rembg”重新合并成一个 Preprocess Service。

## 6. Artifact → Asset → World

| Arrow | Current | Target | Verdict |
|---|---|---|---|
| Provider output → `GenerationOrchestrator` import | 自动 generation path | Caller gets verified Artifact | **REMOVE LEGACY** |
| Caller → AgentScape Asset API | `assetModule.publishAsset({ artifactId, assetId, label })` | stable Artifact → Asset entrance | **MIGRATED 2026-08-28 (`9369e12`)** |
| Asset → World | 已存在 compiler/pipeline | World references reusable Asset ID | **KEEP + PURIFY** |
| WorldRuntime → Provider | 仍有 legacy coupling | 永久禁止 | **REMOVE** |

## 7. modal-gen-client

| Arrow | Current | Target | Verdict |
|---|---|---|---|
| Browser → modal-gen-client | pairing/origin/scope | 保留 | **KEEP** |
| modal-gen-client → Sidecar | 同时带有历史 business routing | mechanical authorization/forwarding only | **SIMPLIFY** |
| Server-side Agent/Hub → modal-gen-client | 可能被当统一中枢 | 可直接调用 Sidecar，不强制 Gateway | **REMOVE AS REQUIREMENT** |

## 8. Kaggle / Embodied / Research

| Arrow | Current | Target | Verdict |
|---|---|---|---|
| Caller → kaggle-inference-hub | AgentScape 主链未形成 canonical adapter | Consumer API Adapter | **VERIFY / ADD AFTER KAGGLE REWRITE** |
| `modal-build → EmbodiedGen` | pinned build/runtime | Build Plane compatibility owner | **KEEP** |
| Provider evidence → AgentScape | legacy adapter 可构造 domain payload | Artifact/Finding → AgentScape admission | **SIMPLIFY** |
| Production Runtime → modal-lab | 无 canonical dependency | 永久保持无 dependency | **KEEP ABSENT** |
| modal-lab → Architecture | 以前主要研究 | experiment evidence 可修改 Card/ADR | **ADD PROCESS** |

## 9. Integration Rule

任何新跨仓箭头必须回答：

```text
Purpose
Caller
Contract Owner
Transport Owner
State Owner
Credential Owner
Retry Owner
Artifact Owner
Failure mapping
```

如果一条箭头需要 Caller 直接访问对方私有 SQLite/ORM/model object，默认判定为架构违规。
