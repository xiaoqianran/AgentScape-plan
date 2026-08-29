# ADR-0008 — AgentScape Convergence：单一 Generation Control Plane

- Status: Accepted
- Date: 2026-08-29

## Context

完成 repository consolidation 与 domain-root layout 后，AgentScape 内部仍残留一套旧时代兼容主链：

```text
LegacyAuthoringShell
→ direct HTTP asset generator
→ source-coded Provider placeholders
→ runtime.authoring
```

Python SDK 同时仍公开 Kaggle/direct Modal Provider client，Studio 的 GenerationJobCenter 仍放在 `generation/`，Agent 的 52 个 Skill handler 与大段 SYSTEM_PROMPT 也集中在单文件中。CI 又按路径过滤，导致跨边界 contract 变化可能漏跑主 Gate。

这些残留不会立即破坏功能，但会让“当前架构”与“实际 public surface”继续分叉。

## Decision

AgentScape 采用单一 provider-neutral Generation control plane：

```text
Studio / Agent / World retry
        │
        ▼
GenerationRuntime
        │
        ▼
Connector session + capability snapshot
        │
        ├─ Job projection / reconcile
        └─ Artifact descriptor / bytes
                │
                ▼
          Asset publication
                │
                ▼
          Asset Compiler / admission
                │
                ▼
          WorldIR / World Runtime
```

### 1. Generation composition

- `GenerationRuntime` 是 AgentScape 内唯一 generation composition root。
- `LegacyAuthoringShell`、`runtime.authoring`、`HttpAssetGenerator` 与 `/api/capabilities/asset-generate` 删除。
- `GenerationRuntime` 可以组合 `GenerationOrchestrator`、Connector、Job/Artifact 与 Asset publication，但不拥有 Provider execution truth。
- Generation domain 不得依赖 Studio。

### 2. Provider discovery

- `ProviderRegistry` 默认不预置任何远程 Provider id。
- `modal-2d`、`modal-3d`、EmbodiedGen 等名称不能作为 AgentScape source-coded default topology。
- 远程 Provider 只能由当前 Connector capability snapshot 动态进入 registry。
- Connector snapshot 消失后，对应 Connector-owned Provider 也必须从 registry 删除。
- 显式 local/test capability 可以注册，但 Connector 不得覆盖其 ownership。

### 3. Studio ownership

`GenerationJobCenter` 是 Human UI component，归 `studio/ui/generation/`，不归 Generation domain。

### 4. Agent capability surface

- 默认 Agent skills 拆成按 domain ownership 的 skill packs。
- `registerCoreSkills.js` 只负责 composition，不保存 handler 实现。
- Provider-specific import tool 不属于默认 Agent surface。
- Agent system prompt 拆成可组合 policy modules；Tool JSON schema/description 是参数与结果 contract 的权威，不再让一个大 SYSTEM_PROMPT 复制完整 Runtime contract。

### 5. Python SDK

Python SDK 只公开 Unified Connector consumer surface：

```text
ConnectorSession
ConnectorCapabilityClient
ConnectorJobClient / Runner
ConnectorArtifactTransport
ConnectorTextTo3DPipeline
request builders / normalized contracts
```

以下 public surface 退役并删除：

```text
agentscape.providers.*
KaggleImageProvider
Modal2DProvider
Modal3DProvider
direct TextTo3DPipeline
reconstruct-direct CLI
```

旧环境变量名只允许作为短期 Connector configuration input alias，不得重新成为 SDK field。

### 6. Tests / CI / tooling

- Tests 按 `agent / asset / generation / world / studio / contracts / integration / e2e` 分组，不再把所有测试平铺到 `tests/`。
- 新增 Convergence validator，机械拒绝旧 surface 回归。
- PR/push 统一执行 AgentScape Runtime Gate、Python SDK Gate、Asset Compiler Service Gate；不得因 path filter 漏掉跨模块 contract 破坏。
- 旧 `repos:*` / submodule sync tooling 删除。

## Consequences

- AgentScape 只有一条正式 generation 主链。
- Provider topology 从 AgentScape source code 移到 Connector-discovered runtime state。
- Python SDK 文档与真实 import surface 一致。
- Studio、Generation、Agent 的 ownership 更清楚。
- 跨 domain 破坏更容易被统一 CI 与 architecture validators 捕获。

## Superseded details

ADR-0004 的 Caller / Capability / Asset / World 分层仍有效，但其中“`GenerationOrchestrator` 不再是目标架构组件”这一实现级结论由本 ADR supersede。

当前允许 `GenerationRuntime / GenerationOrchestrator` 存在于 `AgentScape/generation`，前提是它们只拥有 provider-neutral consumer/orchestration，不拥有 Provider-private execution、credential、GPU lifecycle 或 private artifact storage。

## Verification

AgentScape commit `17d8998` 已验证：

```text
architecture validation   PASS
convergence validation    PASS
world viability           PASS
JS test files             176 PASS
JS tests                  829 PASS
production build          PASS
Python SDK tests          130 PASS
Asset Compiler tests        7 PASS
```
