# ADR-0001 — Repository Purification + Explicit Composition

**Status:** Accepted
**Date:** 2026-08-27

## Decision

AgentScape 多仓库系统采用：

```text
仓库提纯
+
显式跨仓 Contract
+
Artifact-first composition
+
模块化领域核心
```

不采用“一切都由 CapabilityService/Connector/Companion 统一编排”的中心化架构。

## Consequences

- 每个仓库必须有单一稳定身份。
- 不同仓库可以使用 Provider、Queue、Saga、Build Plane、Research 等不同内部架构。
- 迁移采用 Strangler；Legacy Adapter 保持行为，再逐步替换。
- 不做一次性全仓大爆炸 rewrite。
