# ADR-0003 — Provider 与 Local Gateway 的稳定边界

**Status:** Accepted
**Date:** 2026-08-27

## Decision

Provider 拥有特定外部执行领域；Local Gateway 只拥有本机安全/transport 边界。

```text
AgentScape / Caller
   │ requirement
   ▼
Port / Adapter
   ▼
Provider
```

Browser 部署可增加：

```text
Browser → Local Security Gateway → Sidecar/Adapter → Provider
```

## Consequences

- `modal-gen-client` 不拥有 generate_asset workflow。
- `WorldRuntime` 不初始化 Provider/Gateway。
- `modal-2D`、`modal-3D` 保持独立 Provider。
- Sidecar 可以持久化远程执行镜像，但不是业务/Asset truth。
