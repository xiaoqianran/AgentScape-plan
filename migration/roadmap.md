# Architecture Migration Roadmap

目标不是一起重写全部仓库，而是按 **事实验证 → 拓扑纠正 → Capability 稳定 → Caller 建立 → Core 提纯 → Legacy 删除** 迁移。每一步必须有独立 Gate。

# R0 — Architecture Authority

**状态：DONE。**

权威由 `System Landscape / Repository Cards / Shared Contracts / Runtime Views / Integration Ledger / Repository Migrations / Verified Baseline / ADR` 组成。

# R1 — Provider Reality Verification

**状态：DONE。**

```text
040-modal-2d-provider   PASS
041-modal-3d-provider   PASS
```

真实结论：

```text
modal-2D: 2/2 models verified
modal-3D: 4/4 models verified
BiRefNet/current canonicalization verified
Gateway idempotency verified for all 4 models
GLB integrity verified for all 4 models
```

这些结果是后续 3D public-input migration 的 parity baseline。

# R2 — Repository Topology Correction

## R2A — modal-inference-hub

原 GitHub `modal-3D-client` 已原地 rename 为 `modal-inference-hub`，完整历史保留。

目标身份：Human UI / Project / semantic selection / workflow composition。

## R2B — new modal-3D-client

新独立仓已建立并推送，目标身份：纯 3D Reference Sidecar。

## R2C — AgentScape submodule topology

目标：

```text
providers/modal/inference-hub   → modal-inference-hub
providers/modal/object3d-agent  → new modal-3D-client
```

Gate：Hub full tests + new 3D Client tests + Connector/root integration tests。

# R3 — Capability Boundary Stabilization

## R3A — modal-2D / modal-2D-client

**状态：DONE。** Shared Artifact identity、Volume-first fetch、recovery、real GPU Gate 已完成。

## R3B — modal-3D / modal-3D-client

先固定：capability/model identity、Provider Artifact identity、Sidecar durable projection、GLB verification。

随后迁 InputConditioner：

```text
CURRENT public:
canonical 1024 RGBA

TARGET public:
image + optional mask/alpha
        ↓
Provider InputConditioner
        ↓
internal model canonical
```

迁移后必须重跑 `041` 四模型矩阵。

# R4 — Human Caller Purification

`modal-inference-hub`：

```text
H1 Project Domain 保留
H2 Human semantic preprocess 保留
H3 3D execution → new modal-3D-client
H4 image generation → modal-2D-client
H5 candidate workflow 只引用 Sidecar job/artifact
H6 optional publish → AgentScape Asset API
H7 删除 Hub 内重复 Provider Job/Volume implementation
```

Human semantic selection 留 Hub；model-required automatic rembg 下沉 `modal-3D`。

# R5 — AgentScape-agent

**状态：IN PROGRESS。** 独立仓库 `xiaoqianran/AgentScape-agent` 已建立；它是与 `modal-inference-hub` 平级的 Caller，不作为 AgentScape Core 的运行时 submodule 依赖。

```text
A1 Agent Run / tool loop + checkpoint    DONE (resume gated)
A2 AgentScape / 2D / 3D / VLM adapters  PARTIAL (2D+3D real verified)
A3 source_3d_asset Skill                 STARTED
A4 build_world Skill                     TODO
```

当前实现遵循 Experiment-oriented Modular Monolith：生产代码保持 `agent.js + source_3d_asset.js + runs.js` 三个高内聚文件，不建立 service/repository/manager/factory 横向层。`runs.js` 只有因为 Agent Run 具备独立持久化/故障恢复生命周期才被抽出。`source_3d_asset` 使用 Functional Core / Imperative Shell；真实 2D Sidecar adapter 与真实 3D Sidecar adapter 均已验证。3D 路径已通过原始 2D PNG → `modal-3D-client` → Provider InputConditioner/BiRefNet → FastSAM3D++ → verified GLB。2D/3D Adapter 现在都会在首次 Job 前通过 Sidecar `/v1/models` 做 lazy capability preflight，并缓存本次 Adapter capability；未知 model/profile 会在 GPU Job submit 前 fail-closed。VLM adapter 仍只有 contract test，因当前 VPS 没有真实 LLM 凭据而未冒充通过；AgentScape Asset adapter 也仍待接入。Agent step 已支持原子 checkpoint；自动 resume 暂不启用，必须等待 Asset Tool 具备稳定 request identity/idempotency。

旗舰 Gate：

```text
Text
→ N image candidates
→ VLM rank/select
→ 3D
→ Artifact verify
→ Asset Repository
→ World placement
→ Runtime verification
```

# R6 — AgentScape Core Purification

**状态：IN PROGRESS — Asset/World fracture plane 已有 executable evidence。**

| Slice | Status | Evidence |
|---|---|---|
| C1 stable Asset boundary | **DONE** | `9fb7fec` + `9369e12`：`AssetRef`、World Compilation v2、`publishAsset()` public API 已稳定 |
| C2 Asset Repository/state owner | **DONE (module boundary)** | `0a41a93`：AssetManager/Store/Catalog 归 Asset Module；唯一生产构造点为 `createAssetModule()` |
| C3 move Agent/Skills out | **IN PROGRESS elsewhere** | Agent 独立仓已建立；本迁移线不改 Agent |
| C4 caller-driven `publish_asset(Artifact)` | **DONE** | `9369e12`：`assetModule.publishAsset()` 为稳定入口；真实 modal-lab GLB experiment 通过 |
| C5 AssetLibrary → read-only facade | **DONE** | `a0b522a`：generation/provider surface 全部移出；legacy generation 仅在 `LegacyAuthoringShell` |
| C6 remove WorldRuntime → Connector/Generation | **DONE** | `bc3be81`；architecture test fail-closed |
| C7 retire GenerationOrchestrator | **TODO** | 当前仅在 LegacyAuthoringShell |
| C8 ProviderRegistry leaves Asset/World domain | **DONE for domain core** | `7bafc7c`；仅 authoring shell 持有 |
| C9 preserve World Compiler/Runtime behavior | **PASS** | 748/748 root tests + independent Asset/World experiments |

已验证结构：

```text
Asset Module
  createAssetModule()
  → Manager / Store / Catalog
  → AssetRef
       ↓
World Core
  injected Asset module
  → World Compilation v2
  → execution entities contain AssetRef only
```

最终目标不变：AgentScape Core 在零生成 Provider 配置下也能独立运行 Asset/World/Runtime。

# R7 — Kaggle / modal-build Purification

Kaggle：Consumer API / Task Core / Queue+Lease / Worker API / Artifact Layer。

modal-build：Build Plane → immutable runtime artifact → Runtime Plane。

# R8 — Optional Gateway / SDK Cleanup

`modal-gen-client` 收缩成 pairing/origin/scope/credential isolation/mechanical forwarding。

`AgentScape-client` 收缩成 AgentScape Domain SDK/CLI。

# R9 — Legacy Deletion

只在 parity 全绿后删除：

- AgentScape `GenerationOrchestrator`。
- WorldRuntime Provider construction。
- Hub 内旧直接 Modal 3D execution/store。
- Connector 指向旧 3D Hub execution endpoints 的路径。
- duplicate Job truth。
- public canonical-RGBA-only 3D contract（InputConditioner 完成后）。
- Provider payload 直接冒充 ready Asset 的路径。

# 全局迁移规则

1. 一次只改变一个主要 Ownership。
2. Strangler：新边界先接 Legacy Adapter，parity 后删除旧实现。
3. 破坏性 Contract 迁移前先保存真实 baseline experiment。
4. 两个实现不自动意味着要建 framework。
5. 当 identity/state/input-output/failure/retry/smoke/contract 都清晰时停止继续拆。
