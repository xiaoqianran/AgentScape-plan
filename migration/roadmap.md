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

新建独立仓库。

```text
A1 Agent Run / checkpoint
A2 AgentScape / 2D / 3D / VLM adapters
A3 source_3d_asset Skill
A4 build_world Skill
```

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

```text
C1 stable Asset API
C2 Asset Repository
C3 move Agent/Skills to AgentScape-agent
C4 caller-driven publish_asset(Artifact)
C5 AssetLibrary → repository only
C6 remove WorldRuntime → ConnectorClient
C7 retire GenerationOrchestrator
C8 ProviderRegistry leaves domain path
C9 preserve World Compiler/Runtime behavior
```

最终：AgentScape Core 在零生成 Provider 配置下也能独立运行 Asset/World/Runtime。

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
