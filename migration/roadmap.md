# Consolidation Roadmap

当前目标不是继续拆仓，而是完成收敛后的清理。

## R1 — Documentation authority

- [x] System Landscape 改为 `AgentScape + modal-provider`。
- [x] Repository Cards 删除 14-repo 当前态。
- [x] 增加 ADR-0006。
- [x] Integration Ledger 重写为 merge/retire 映射。

## R2 — Active documentation cleanup

- [ ] AgentScape active docs 不再把旧 Provider package 写成独立 repo。
- [ ] `modal-provider` README 明确 monorepo ownership。
- [ ] package README 中旧 Hub/standalone-repo 语言清理。

## R3 — Tooling cleanup

- [ ] 移除已经无意义的跨仓 submodule/workspace 假设。
- [ ] CI 使用 `modal-provider` 内 package matrix/path filters。
- [ ] release/deploy 仍可按 package 独立执行。

## R4 — Verification gate

必须持续通过：

```text
AgentScape architecture validation
AgentScape targeted provider/connector tests
AgentScape full test/build gate
modal-provider package unit tests
provider deploy/smoke where credentials are available
```

## Stop condition

当 active docs、CI、开发命令都只需要理解三个仓库：

```text
AgentScape
modal-provider
AgentScape-plan
```

且运行时只需要前两个仓库时，本轮仓库收敛完成。
