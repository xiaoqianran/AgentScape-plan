# AgentScape Execution Task Spec 模板

> 用于把 `04-live-execution-map.md` 中的一个可执行切片交给单个 AI/开发者。一个 Task Spec 必须足够独立，使执行者不需要从聊天记录猜目标，也不能把“计划”误报成“完成”。

## 1. Task Identity

```text
Task ID:        <例如 C-01>
Parent Plan:    <AS-02 / Gate L2>
Owner Track:    <Runtime Truth / Provider / Connector / Artifact / World / Backend>
Status:         PLANNED | IN_PROGRESS | COMMITTED_NOT_MERGED | MERGED | BLOCKED
Base Commit:    <开始实施时的 AgentScape HEAD>
Feature Branch: <branch name>
```

## 2. Goal

一句话描述本任务交付的**可验证能力**，避免写“优化”“完善”“支持更多”这类不可验收表述。

示例：

> 建立 Local Connector pairing session contract，使 browser 只能持有 scoped/expiring session identity，不能读取 provider credential；AgentScape 能通过该 session 获取 versioned capability snapshot。

## 3. Non-goals

明确本任务不做什么。至少列出相邻但后置的能力，防止 scope creep。

示例：

- 不实现 Async Job DB；
- 不实现 Job Center UI；
- 不接真实 Modal GPU；
- 不修改 Physics/Interaction；
- 不把 provider capability 升级成 Runtime action truth。

## 4. Preconditions / Gate

任务开始前必须满足：

- [ ] 所依赖 Gate 已通过；
- [ ] AgentScape `git status` 已读取；
- [ ] 没有与本任务 ownership 冲突的 dirty WIP；
- [ ] base commit 已记录；
- [ ] 相关 stash/feature branch 已确认不会被覆盖；
- [ ] 必要时 CodeGraph 已 `sync`。

如果任一硬前置不满足，任务状态写 `BLOCKED`，不要边做边赌 merge。

## 5. Current Code Facts

只记录实际读取到的事实：

```text
Relevant symbols:
Relevant files:
Current callers/callees:
Current tests:
Known compatibility path:
Known debt/risk:
```

不得把 roadmap 里的目标接口写成“当前已有”。

## 6. CodeGraph Impact Baseline

实施前至少记录：

```text
symbol / module:
impact depth:
nodes:
edges:
src files:
test files:
```

如果任务触碰 `PhysicsSystem`、`InteractionSystem`、`WorldRuntime`、`ToolCallingAgent` 等高 blast-radius 节点，必须说明为何不能用外围 adapter/facade 完成。

## 7. Ownership

### Allowed primary files

```text
<预计新增/修改文件>
```

### Avoid / forbidden concurrent files

```text
<其它 track 正在修改或高风险文件>
```

若实施过程中发现必须跨 ownership，先暂停并更新 Live Map，不偷偷扩大 patch。

## 8. Contract Invariants

列出即使实现方式变化也不能破坏的语义。

AgentScape 常见不变量：

```text
Evidence != Proposal != Executable != Verified
Provider succeeded != Asset ready
Recovery success != Original task success
Remote URL != Artifact identity
LLM semantic constraint != Runtime placement truth
Generated bundle != Runtime capability
pending job != completed task
world rollback != artifact deletion
```

任务至少选择与自身有关的不变量，并补充专属 contract。

## 9. Proposed Change

按“最小稳定接口”描述，不要求预先锁死内部实现：

```text
new module(s):
public interface:
input contract:
output contract:
error/status contract:
persistence/provenance:
compatibility behavior:
```

优先新增窄接口，避免一次重构多个既有系统。

## 10. Failure Semantics

必须显式回答：

- unavailable 怎么表示？
- timeout 怎么表示？
- retry 是否安全？
- duplicate request 怎么处理？
- partial success 怎么处理？
- rollback 范围是什么？
- 哪些结果只能作为 evidence？
- 用户/Agent 是否会被误导成 completed？

## 11. Security / Trust Boundary

根据任务选择：

- secret 是否可能进入 browser/log/scene？
- origin/session/scope 如何约束？
- remote URL 是否可信？
- hash/MIME/length 是否验证？
- provider semantic evidence 的 trust level？
- path/redirect/bundle traversal 风险？

不适用的项明确写 N/A，不要省略整节。

## 12. Test Plan

至少分三层：

### Contract tests

- happy path；
- unavailable/disabled；
- malformed input/output；
- duplicate/idempotency；
- stale/revoked/version mismatch（适用时）。

### Regression tests

列出必须保持通过的既有 tests。

### Integration / E2E

明确是否需要真实 provider、fixture、browser 或 WorldRuntime。

## 13. Verification Commands

写实际要执行的命令，而不是“运行测试”：

```text
<focused tests>
<related suite>
<asset validation>
<production build>
<full regression if required>
```

规则：只有命令 `exit_code=0` 才能记录 PASS；timeout 单独写 TIMEOUT。

## 14. Definition of Done

- [ ] 实现完成；
- [ ] contract tests PASS；
- [ ] regression PASS；
- [ ] build 有明确结果；
- [ ] 无意外 ownership 扩散；
- [ ] feature branch/commit 已记录；
- [ ] Live Map 状态已更新；
- [ ] 若稳定 contract 改变，已回写 `01/02/03` 或 cross-cutting 文档；
- [ ] 下一 Gate 是否解锁已明确。

## 15. Completion Evidence

完成时填写：

```text
Final Status:
Commit / PR:
Merged into main:
Files changed:
CodeGraph impact after:
Focused tests:
Full tests:
Build:
Known warnings:
Known follow-up:
Unlocked Gate:
```

## 16. Handoff to Next Task

只写**下一项已经满足前置条件**的任务。如果仍有 blocker，写 blocker，不提前宣布进入下一阶段。
