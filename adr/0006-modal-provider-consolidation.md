# ADR-0006 — Consolidate Provider Repositories into `modal-provider`

- Status: Accepted
- Date: 2026-08-29

## Context

早期 AgentScape 为了快速验证 ownership，把 Agent、Human workflow、Gateway、2D/3D Provider、Sidecar、Kaggle、Build、EmbodiedGen integration、Lab 分散成多个仓库。随着契约稳定，这种 repository topology 开始放大同步成本，并让文档错误地把“package/deployment unit”当成“系统级 repository authority”。

## Decision

1. Agent/Human/Asset/World 业务能力收敛到 `AgentScape`。
2. Modal Provider 相关能力收敛到 `modal-provider` monorepo。
3. `modal-provider` 内保留独立 package、lockfile、测试、Modal app 和 deployment unit；这些不再是独立仓库。
4. `modal-EmbodiedGen` 接管 EmbodiedGen build/runtime integration；上游 EmbodiedGen 仅作为 pinned source dependency。
5. `kaggle-inference-hub` 与独立 `modal-lab` 不再属于目标架构。
6. `AgentScape-plan` 继续作为文档权威，但不进入 runtime。

## Consequences

### Positive

- repository ownership 与 runtime ownership 更一致；
- Provider 原子变更可在一个 monorepo commit 中完成；
- 跨 Provider contract test 更容易；
- 不再需要维护大量 submodule/repo pin；
- 文档只需描述两个产品仓库。

### Constraints

- 不能因为 monorepo 合并就消除 Provider/Sidecar/Gateway 的运行时边界；
- 不能让 AgentScape import `modal-provider` 私有实现；
- package 独立部署、失败、凭据与 GPU 生命周期必须继续隔离；
- 历史 ADR/benchmark 中旧仓库名称只具有历史含义。

## Supersedes

本 ADR **只在 repository topology 层面** supersede 早期多仓库假设。ADR-0003/0004/0005 中关于安全边界、Caller/Capability/Asset/World 分层、按压力拆分的原则仍有效。
