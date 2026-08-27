# ADR-0002 — Execution / Artifact / Asset / World 分离

**Status:** Accepted
**Date:** 2026-08-27

## Decision

```text
Execution = 可变执行过程
Artifact  = 不可变内容事实
Asset     = AgentScape 语义对象
World     = Desired/Compiled/Observed world state
```

四者禁止合并成统一 Job/Result 模型。

## Consequences

- Provider Job 成功不能自动产生 ready Asset。
- Artifact 必须支持 digest/bytes/mediaType 独立验证。
- Provider evidence 不能声明 AgentScape Runtime 已验证。
- Job ID 不需要全局统一。
