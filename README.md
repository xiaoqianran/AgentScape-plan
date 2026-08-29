# AgentScape Architecture Plan

本仓库只维护 **AgentScape 当前架构权威与迁移记录**。它不是运行时组件，也不再维护旧的“十几个独立仓库”拓扑。

> 2026-08-29 起，AgentScape 的产品代码边界收敛为 `AgentScape` + `modal-provider`。原先分散的 Agent、Human Workflow、Provider、Sidecar、Build/Embodied Runtime 等仓库，要么并入这两个主仓，要么退役。

## 当前仓库边界

| Repository | 当前职责 |
|---|---|
| `AgentScape` | Agent/Studio、GenerationRuntime、Connector/Job/Artifact projection、Asset、World、Runtime/Verification、Connector-only Python SDK |
| `modal-provider` | Modal 本地安全网关、2D/3D Provider、Reference Sidecar、EmbodiedGen build/runtime integration |
| `AgentScape-plan` | 架构决策、迁移记录、验证基线；不参与运行时 |

`modal-provider` 内部仍可拥有多个 package / deployment unit，但这些是 **monorepo 内部边界，不是独立系统仓库边界**。

## 已退役的独立仓库概念

以下名称不得再出现在“当前仓库拓扑”中：

- `AgentScape-agent`
- `modal-inference-hub`
- `modal-gen-client`
- `modal-2D`
- `modal-2D-client`
- `modal-3D`
- `modal-3D-client`
- `kaggle-inference-hub`
- `modal-build`
- `EmbodiedGen` 独立 checkout
- `modal-lab`

其中 Provider 相关实现已收敛进 `modal-provider`；Agent/Human 工作流已收敛进 `AgentScape`；Kaggle 与独立实验仓不再构成目标运行时架构。

## 权威顺序

1. [`architecture/system-landscape.md`](architecture/system-landscape.md) — 当前系统边界与 ownership。
2. [`architecture/repository-cards.md`](architecture/repository-cards.md) — 三个当前仓库及 `modal-provider` 内部 package card。
3. [`architecture/shared-contracts.md`](architecture/shared-contracts.md) — Capability、Execution、Artifact、Finding、Asset、World 契约。
4. [`architecture/runtime-views.md`](architecture/runtime-views.md) — 关键运行链路与失败所有权。
5. [`architecture/runtime-backend-plane.md`](architecture/runtime-backend-plane.md) — Physics/Navigation deep backend contract、solver ownership 与 evidence quality。
6. [`migration/integration-ledger.md`](migration/integration-ledger.md) — 旧拓扑到新拓扑的收敛账本。
7. [`migration/repository-migrations.md`](migration/repository-migrations.md) — 仓库合并/退役映射。
8. [`migration/roadmap.md`](migration/roadmap.md) — 后续清理 Gate。
9. [`status/verified-baseline.md`](status/verified-baseline.md) — 当前已验证基线。
10. [`adr/0006-modal-provider-consolidation.md`](adr/0006-modal-provider-consolidation.md) — 本轮仓库收敛决策。
11. [`adr/0007-domain-root-repository-layout.md`](adr/0007-domain-root-repository-layout.md) — AgentScape domain-root 目录与依赖门禁。
12. [`adr/0008-agentscape-convergence.md`](adr/0008-agentscape-convergence.md) — 单一 GenerationRuntime、Connector-only Provider discovery、SDK/Skill/CI 收敛。
13. [`adr/0009-observatory-developer-surface.md`](adr/0009-observatory-developer-surface.md) — Observatory Developer Surface、单向依赖与 Runtime debug contract。
14. [`adr/0010-replaceable-runtime-backends.md`](adr/0010-replaceable-runtime-backends.md) — Physics/Navigation replaceable backend ownership、native leakage 与 conformance/parity Gate。

## 不变量

```text
Repository topology != runtime truth
Package boundary      != repository boundary
Provider output       != Asset truth
Asset truth           != World truth
Human caller          == Agent caller at capability boundary
Observatory            != business truth owner
```

任何新文档若重新把 `modal-2D`、`modal-3D`、各 client 或 `modal-build` 写成独立仓库，都属于架构回退。
