# ADR-0004 — Caller / Capability / Asset / World 分离

> **Repository-topology note (2026-08-29):** Caller/Capability/Asset/World 分层仍有效；`AgentScape-agent` / `modal-inference-hub` 作为独立仓库的决策已由 [ADR-0006](./0006-modal-provider-consolidation.md) supersede，相关能力现归 `AgentScape`。


**Status:** Accepted
**Date:** 2026-08-27

## Context

原架构仍把 Agent、GenerationOrchestrator、Provider 与 WorldRuntime 放在同一个聚合核心附近；旧 `modal-3D-client` 同时拥有 Human Project、Preprocess、Provider execution 和 Artifact transport。

真实需求已经明确出现两类平级 Caller：

```text
AgentScape-agent    = Agent/VLM 自动工作流
modal-inference-hub = Human/UI 手工工作流
```

二者都需要调用同一组稳定 2D/3D capability，并把结果发布为 AgentScape Asset/World。

## Decision

系统按四层分离：

```text
Caller       → decides intent / selection / workflow
Capability   → generates/transforms content
Asset        → owns reusable semantic object
World        → owns placement/relations/runtime
```

具体决策：

1. Agent 从 `AgentScape` Core 移出，建立独立 `AgentScape-agent`。
2. 原 `modal-3D-client` 演化为 `modal-inference-hub`，成为 Human Caller。
3. 新建纯 `modal-3D-client` Reference Sidecar，与 `modal-2D-client` 对称。
4. `AgentScape-agent` 与 `modal-inference-hub` 平级，没有唯一 Provider 上级。
5. 默认 Text→3D 是 Caller Skill：Text → image candidates → selection → Image→3D，而不是伪造 Provider capability。
6. AgentScape Core 只拥有 Asset + World，不拥有生成 Provider workflow。
7. model-required background removal/crop/normalize 归 `modal-3D InputConditioner`；Human/Agent semantic selection 归 Caller。

## Consequences

- `GenerationOrchestrator` 不再是目标架构组件。
- WorldRuntime 最终完全不知道 Connector/Provider。
- Hub 与 Agent 可以共享 Sidecar/Provider contract，但不共享 Project/AgentRun state。
- `modal-3D-client` public input 最终不应要求 Caller 理解模型私有 canonical 规则。
- reusable Asset 与 World Instance placement 永久分离。
- Provider migration 必须先有真实 `modal-lab` baseline，再做破坏性 contract 迁移。
