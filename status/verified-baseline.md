# Verified Baseline

本文只记录迁移后仍需保持的能力，不再把旧仓库数量当成能力指标。

## Repository baseline — 2026-08-29

```text
AgentScape        product/domain/runtime monorepo
modal-provider    Modal provider monorepo
AgentScape-plan   architecture documentation authority
```

AgentScape 当前架构校验明确要求：Provider repository **不得**再以 submodule 形式 pin 到 AgentScape。

## Capability baseline

已建立并必须保持的能力包括：

- provider-neutral capability snapshot / registry；
- Connector session、Job、Artifact projection；
- 2D Provider + Sidecar 的真实 PNG 生成/校验路径；
- 3D Provider + Sidecar 的真实 GLB 生成/校验路径；
- 2D candidate batch 与 3D generation composition；
- AgentScape Artifact integrity gate；
- Asset Compiler / Asset admission；
- generated-world admission、deterministic composition 与 Runtime verification；
- EmbodiedGen raw payload → Adapter → Manifest/admission 路径；
- ToolCallingAgent / Skill / Agent recovery 测试链。

## Consolidation rule

合并仓库不得改变这些契约语义：

```text
Provider-private execution stays provider-private
Artifact integrity stays verifiable
Asset admission stays AgentScape-owned
World truth stays AgentScape-owned
```

旧仓库名可继续出现在历史 commit、benchmark、实验编号中，但不得被解释为当前系统 topology。
