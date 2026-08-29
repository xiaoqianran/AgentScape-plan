# Verified Baseline

本文记录 2026-08-29 AgentScape Convergence 后必须保持的当前能力与工程 Gate，不再以旧仓库数量衡量系统成熟度。

## Repository baseline

```text
AgentScape        product/domain/runtime monorepo
modal-provider    Modal provider monorepo
AgentScape-plan   architecture documentation authority
```

AgentScape main baseline：

```text
commit 1bf17a6
feat: add runtime observatory
```

Provider repository 不以 submodule 形式 pin 到 AgentScape；旧 `repos:*` / recursive submodule tooling 已删除。

## Product/domain baseline

AgentScape 当前六个 business product/domain roots：

```text
studio/
agent/
generation/
asset/
world/
core/
```

Developer Product Surface：

```text
observatory/
```

Observatory 单向消费 production Runtime/domain，不拥有业务 truth。

真实 engineering/release boundaries：

```text
api/
services/
sdk/
tests/
tooling/
docs/
public/
```

## Generation baseline

必须保持：

- `GenerationRuntime` 是唯一 generation composition root；
- Connector session + capability snapshot；
- ProviderRegistry 默认不硬编码远程 Provider；
- Connector-owned Provider 随 snapshot 进入/退出 registry；
- Job projection / reconcile / cancel；
- Artifact descriptor、bytes、digest/integrity gate；
- Artifact → Asset publication；
- Asset Compiler / admission；
- `LegacyAuthoringShell`、`runtime.authoring`、direct asset generator 不得恢复。

## Agent baseline

必须保持：

- ToolCallingAgent mutation identity / fresh replan / recovery guard；
- domain skill packs；
- `registerCoreSkills.js` composition-only；
- composable prompt policies；
- Tool schema/description 是参数与结果 contract authority；
- Provider-specific import tool 不属于默认 Agent surface。

## Asset / World baseline

必须保持：

- Artifact integrity gate；
- Asset Compiler / Asset admission；
- reusable Asset truth；
- WorldIR / deterministic composition；
- canonical `ON / NEAR / INSIDE` placement semantics；
- Physics / Navigation / Interaction；
- Runtime Verification / Acceptance；
- World viability benchmark；
- generated-world rollback / bounded retry / revision lineage。


## Observatory baseline

必须保持：

- independent `/observatory/` Vite entry；
- Physics Lab：Rapier/Jolt、normalized comparison、fixed-step replay；
- Spatial Lab：BVH raycast、bounds/overlap、support/free-space；
- `PhysicsSystem.debugSnapshot()` / `SpatialSystem.debugSnapshot()` observation boundary；
- production code 不依赖 Observatory；
- synthetic fixture 不复制 production Manifest；
- Observatory tests 位于 `tests/observatory/`。

## Python SDK baseline

Python SDK 是 Unified Connector consumer。

公开目标 surface：

```text
ConnectorSession
ConnectorCapabilityClient
ConnectorJobClient / Runner
ConnectorArtifactTransport
ConnectorTextTo3DPipeline
request builders / normalized contracts
```

退役 public surface 不得恢复：

```text
agentscape.providers.*
KaggleImageProvider
Modal2DProvider
Modal3DProvider
direct TextTo3DPipeline
reconstruct-direct CLI
```

## Tests baseline

Committed AgentScape tests 已按 ownership/scope 分组：

```text
tests/agent
tests/asset
tests/generation
tests/world
tests/studio
tests/observatory
tests/contracts
tests/integration
tests/e2e
```

不得重新把主测试体系平铺回 `tests/` 根目录。

## CI baseline

`.github/workflows/agentscape-check.yml` 在 push / pull_request / workflow_dispatch 上统一执行：

```text
Runtime / World / Studio
  → npm ci
  → npm run check

Python SDK
  → uv sync --frozen --extra dev
  → pytest / compileall / build

Asset Compiler Service
  → install test requirements
  → unittest
```

主 Gate 不使用 path filter，因此跨模块 contract 变化不能因为“没改那个目录”而漏测。

## Verified Gate — AgentScape 1bf17a6

```text
architecture validation   PASS
convergence validation    PASS
world viability           PASS
JS test files             180 PASS
JS tests                  858 PASS
production build          PASS
Python SDK tests          130 PASS
Asset Compiler tests        7 PASS
git diff --check          PASS
```

## Global invariants

```text
Provider-private execution stays provider-private
Remote Provider topology is runtime-discovered, not source-coded
Artifact integrity stays verifiable
Asset admission stays AgentScape-owned
World truth stays AgentScape-owned
Prompt text does not replace structured Tool contracts
```

旧仓库名、旧 direct Provider API 与 LegacyAuthoring 可以继续出现在 historical ADR / migration record 中，但不得被解释为当前 runtime surface。
