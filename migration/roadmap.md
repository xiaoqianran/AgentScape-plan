# Consolidation Roadmap

当前阶段不是继续拆仓，而是保证收敛后的单一主链不会再次长出旧兼容架构。

## R1 — Repository authority

- [x] System Landscape 收敛为 `AgentScape + modal-provider`。
- [x] Repository Cards 删除旧多仓当前态。
- [x] ADR-0006 固化 repository consolidation。
- [x] ADR-0007 固化 AgentScape domain-root layout。
- [x] ADR-0008 固化 Generation/SDK/Skill/CI convergence。

## R2 — AgentScape runtime convergence

- [x] `LegacyAuthoringShell` 删除。
- [x] `runtime.authoring` 删除。
- [x] direct HTTP asset generator 删除。
- [x] `/api/capabilities/asset-generate` 删除。
- [x] `GenerationRuntime` 成为唯一 generation composition root。
- [x] ProviderRegistry 删除 source-coded remote Provider placeholders。
- [x] Connector snapshot 成为远程 Provider discovery truth。
- [x] `GenerationJobCenter` 移入 Studio UI ownership。

## R3 — Agent / SDK convergence

- [x] 52 个 Skill handler 拆为 domain skill packs。
- [x] `registerCoreSkills.js` 收敛为 composition entry。
- [x] 大 SYSTEM_PROMPT 拆为 composable prompt policies。
- [x] 默认 Agent surface 删除 Provider-specific import tool。
- [x] Python SDK 删除 Kaggle/direct Modal Provider clients。
- [x] Python SDK 删除 direct generation pipeline / `reconstruct-direct`。
- [x] SDK README 与实际 public surface 对齐。

## R4 — Tests / CI / tooling

- [x] 主测试体系按 domain / contracts / integration / e2e 分组。
- [x] 新增 Convergence validator。
- [x] 删除 AgentScape `repos:*` / recursive submodule tooling。
- [x] 新增无 path-filter 的统一 AgentScape PR/push Gate。
- [x] Runtime、Python SDK、Asset Compiler Service 都进入统一 CI workflow。

## R5 — Provider monorepo maintenance

以下属于 `modal-provider` 自己的 package-level 持续维护，不要求恢复 repository split：

- [ ] 持续维护 package-level unit/deploy/smoke matrix。
- [ ] 有 credential 时持续跑真实 Provider smoke。
- [ ] 保持 package 独立 lockfile/runtime/deploy 生命周期。

## Continuous verification

AgentScape 必须持续通过：

```text
architecture validation
convergence validation
asset validation
world viability
full JS tests
production build
Python SDK tests/build
Asset Compiler Service tests
```

modal-provider 必须在自己的仓库内保持对应 package test/deploy/smoke。

## Stop condition

Repository consolidation 与 AgentScape internal convergence 已完成。

后续任何新能力都必须满足：

```text
new capability
  ≠ restore old repo boundary
  ≠ restore direct Provider client
  ≠ add source-coded remote Provider topology
  ≠ bypass Artifact/Asset/World admission
```

新的架构变化应通过新的 ADR 演进，而不是恢复已退役兼容层。
