# AgentScape Architecture Plan

本仓库只维护 **AgentScape 多仓库系统的架构权威与迁移计划**。

旧计划已从当前工作树移除；历史仍完整保存在 Git。不要为了兼容旧 PA/P15/P19/Studio/Companion 路线重新引入平行权威。

## 权威顺序

1. [`architecture/system-landscape.md`](architecture/system-landscape.md) — 系统边界、仓库角色、全局不变量。
2. [`architecture/repository-cards.md`](architecture/repository-cards.md) — 14 个仓库的目标 Architecture Card；仓库重写以此为目标。
3. [`architecture/shared-contracts.md`](architecture/shared-contracts.md) — 跨仓库稳定语义：Capability、Execution、Artifact、Finding、Asset、World。
4. [`architecture/runtime-views.md`](architecture/runtime-views.md) — 关键运行链路与状态/失败所有权。
5. [`migration/integration-ledger.md`](migration/integration-ledger.md) — 当前真实箭头、目标箭头、KEEP/MOVE/SIMPLIFY/REMOVE 判定。
6. [`migration/repository-migrations.md`](migration/repository-migrations.md) — 每个仓库的 Current → Target 迁移切片。
7. [`migration/roadmap.md`](migration/roadmap.md) — 全局执行顺序、Gate 与停止条件。
8. [`status/verified-baseline.md`](status/verified-baseline.md) — 当前已真实验证的能力基线；迁移不得回退。
9. [`adr/`](adr/) — 只有 architecture-significant decision 才记录 ADR。

## 计划原则

```text
先定 Ownership
    ↓
再定 Contract
    ↓
再定 Integration
    ↓
再迁移代码
    ↓
每个切片独立验证
    ↓
最后删除 Legacy
```

默认实现策略由 [`ADR-0005`](adr/0005-experiment-oriented-modular-monolith.md) 约束：**Experiment-oriented Modular Monolith → Vertical Slice → Single-file First → Functional Core / Imperative Shell → Extract by Pressure**。

默认不拆文件。只有出现独立 State Owner、failure/retry 生命周期、GPU/部署生命周期、安全边界、测试矩阵、持续冲突或真实性能压力时才拆。行数不是拆分依据。

## 架构完成标准

任何仓库宣布迁移完成前必须满足：

```text
[ ] 一句话说清仓库身份
[ ] 明确 Does Not Own
[ ] State Owner 少且清晰
[ ] 最小输入稳定
[ ] 最小输出稳定
[ ] failure/retry owner 明确
[ ] 可脱离 AgentScape 独立 smoke
[ ] Artifact/Contract 可独立验证
[ ] 上游不依赖私有实现
[ ] 没有新增无意义抽象层
[ ] Integration Ledger 已更新
[ ] 依赖图符合 Architecture Card
```
