# Architecture Migration Roadmap

目标不是“所有仓库一起重写”，而是**逐个稳定边界，保持真实能力持续工作，再删除 Legacy**。

# R0 — Architecture Authority

**状态：本次重建完成后 DONE。**

交付：

```text
System Landscape
Repository Cards
Shared Contracts
Runtime Views
Integration Ledger
Repository Migrations
Verified Baseline
ADRs
```

Gate：当前工作树只存在一套架构权威，不再有 PA/P15/P19/Studio/Companion 平行路线。

# R1 — Integration Truth Freeze

目标：在改生产代码前，把所有跨仓真实箭头补齐到 Integration Ledger。

任务：

1. 对每条跨仓调用确认 endpoint/contract/credential/state/retry/artifact owner。
2. 当前没有 canonical runtime arrow 的仓库明确标 `ABSENT/VERIFY`，不脑补。
3. 每个 Production Provider 固化独立 smoke 命令。
4. Shared Contracts 只冻结语义，不创建新的 shared-contract repo/package。

Gate：所有 production 箭头都能回答 `Purpose / Contract / State Owner / Retry Owner / Artifact Owner`。

# R2 — Low-risk Provider Boundary Stabilization

顺序：

```text
modal-2D
  ↓
modal-2D-client
  ↓
modal-3D
```

这些仓库边界已较健康，只做 contract/readiness/artifact/smoke 对齐，不做大 rewrite。

Gate：

- 2D real GPU → PNG verify。
- 2D sidecar restart recovery。
- 每个 enabled 3D model → GLB verify。
- 3D capability/readiness cache/cold-start 行为稳定。

# R3 — Deep Repository Purification

三个高风险仓库**串行迁移，不并行改跨仓 Contract**。

## R3A — modal-3D-client

迁移为：

```text
Project
Preprocess
Generation Saga
+ Application Boundary
+ 外围 Connector/UI Adapter
```

完成后再进入 R3B。

Gate：raw image → preprocess → real 3D → GLB + restart recovery + Connector compatibility。

## R3B — kaggle-inference-hub

迁移为：

```text
Consumer API
Task Core / Queue / Lease
Worker API
Artifact Layer
外围 Prompt/UI
```

Gate：worker crash/reclaim、consumer isolation、artifact verification。

## R3C — modal-build

迁移为：

```text
Build Plane
    ↓ immutable runtime artifact
Runtime Plane
```

并按 GPU lifecycle 拆分大型 runtime。

Gate：可复现 build、无动态 CUDA build、各 capability real GPU smoke、evidence 分层不回退。

# R4 — AgentScape Core Purification

只有 Provider/Sidecar 边界稳定后才动聚合核心。

顺序：

```text
Composition Root
   ↓
Asset Sourcing Port + Legacy Adapter
   ↓
Artifact-first verify/compile
   ↓
AssetLibrary shrink
   ↓
Catalog shrink
   ↓
GenerationOrchestrator retire
   ↓
Skill surface cleanup
```

每个切片必须保持 World/Agent 回归。

最终 Gate：

```text
WorldRuntime 不引用 Connector/Provider 私有实现
WorldPipeline 只消费 Asset / World IR
Provider success 不跳过 Artifact/Asset/World verification
```

# R5 — Gateway / Client Simplification

## modal-gen-client

收缩为 Security + Transport：pairing/origin/scope/secret isolation/adapter forwarding/local projection。

## AgentScape-client

收敛为 Reference SDK/CLI + Artifact verifier，支持 direct/HTTP/gateway transport。

Gate：同一 Provider 可通过 reference client 独立验证；Gateway 不拥有业务 composition。

# R6 — New Provider Admission

Kaggle/Embodied/未来 Provider 进入 AgentScape 时统一走：

```text
Independent Provider Smoke
    ↓
Capability Descriptor
    ↓
Adapter
    ↓
Execution Projection
    ↓
Artifact Verify
    ↓
Asset Compiler
    ↓
AgentScape E2E
```

禁止直接给 WorldRuntime 加 Provider 特例。

# R7 — Legacy Deletion

只有 parity 证据充分后删除：

- Legacy generation control plane。
- 重复 Job truth / Store。
- Connector 对 Domain Store 的 shortcut。
- Provider payload 直接构造 verified Asset/Manifest 的路径。
- 旧 full-build runtime path。

删除条件：对应迁移 Gate 全绿 + 无线上/真实 canary 依赖 Legacy。

# R8 — Thin Evidence UI（后置）

UI 首先展示：

```text
Agent decision
Capability binding
Execution
Artifact lineage
Verification findings
Asset admission
World findings
```

不要先做复杂 Studio/资产产品。

# 全局迁移规则

## 一次只改变一个 Ownership

一个切片可以改很多行，但只能改变一个主要 State Owner/边界。

## Strangler

```text
New Port
   ├─ Legacy Adapter → old implementation
   └─ New implementation
```

先 parity，再删 old。

## 不提前抽象

两个实现不自动意味着要建 framework。只有稳定差异与重复 contract 被真实证明后才抽象。

## 重构停止条件

如果一个仓库已经：

```text
identity 清晰
state owner 清晰
独立 smoke 清晰
跨仓 contract 清晰
```

即使内部文件不“漂亮”，也停止继续拆。
