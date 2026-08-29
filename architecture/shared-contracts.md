# Shared Contracts

本文件只定义跨仓库/跨 runtime boundary 需要稳定的最小语义。禁止为了“统一”而复制所有 Provider 私有参数。

# 1. Capability Snapshot

AgentScape 通过 Connector snapshot 发现远程能力，而不是在源码中硬编码 Provider topology。

```text
CapabilitySnapshot
├─ connector
│  ├─ id
│  ├─ instance
│  └─ version
├─ revision
├─ hash
└─ providers[]
   ├─ id
   ├─ status / health
   ├─ artifactTransport
   └─ capabilities[]
      ├─ operation
      ├─ category
      ├─ input
      ├─ output
      ├─ execution
      └─ support
```

规则：

- remote Provider id 只来自当前 snapshot；
- ProviderRegistry 不预置远程 placeholder；
- Connector-owned Provider 随 snapshot 生命周期进入/退出 registry；
- local/test provider 可以显式注册，但 Connector 不得抢占 ownership；
- snapshot 是消费者 projection，不是 Provider 内部 truth 的完整复制。

# 2. Capability Descriptor

```text
CapabilityDescriptor
├─ provider
├─ operation
├─ category
├─ status
├─ input
├─ output
├─ execution
├─ support
└─ artifactTransport
```

规则：

- `operation` 必须是稳定 provider-scoped id；
- capability selection 只消费稳定字段；
- Provider 私有 model/profile/options 仍在 Provider namespace 内；
- Catalog/Registry 做 register/list/find/resolve，不拥有 Provider execution。

# 3. Job / Execution Projection

```text
GenerationJobProjection
├─ requestId
├─ jobId
├─ provider
├─ operation
├─ state
├─ createdAt
├─ updatedAt
├─ failure?
└─ artifacts[]
```

规则：

- Provider 可以有更丰富内部状态；AgentScape 只保存稳定投影；
- submit timeout 不能自动等价于 failed；
- request identity 必须支持幂等绑定；
- Job projection 不是 Provider-private Job database。

# 4. Artifact Descriptor

```text
ArtifactDescriptor
├─ id
├─ role
├─ mediaType
├─ bytes?
├─ digest
├─ producer
│  ├─ provider
│  ├─ operation
│  └─ revision?
└─ lineage[]
```

规则：

- Content identity 与 location 分离；
- digest 对应内容不可原地覆盖；
- 派生处理产生新 Artifact；
- private Volume path / Modal call id 不成为 AgentScape contract；
- Artifact 进入 Asset publication 前必须经过本地 integrity/content gate。

# 5. Asset

Asset 是 AgentScape Domain Object，不是 Provider Artifact 的别名。

```text
Asset
├─ assetId
├─ semantic type / tags
├─ actions
├─ physics
├─ colliders
├─ surfaces
├─ receptacles?
├─ quality / admission
└─ sourceArtifactLineage
```

只有 Asset Compiler / Admission 可以决定：

```text
ready
provisional
rejected
```

Provider 不得直接声明 AgentScape Asset 已验证。

## 5.1 AssetRef

World execution identity 只通过：

```text
AssetRef
└─ assetId
```

`query / prompt / generate / provider` 属于 Caller/WorldIR asset request，不是 live World entity identity。

当前实现锚点：

```text
asset/AssetRef.js
generation/orchestration/createAssetModule.js
generation/orchestration/GenerationRuntime.js
```

# 6. World

```text
WorldIR        = desired intent + request policy
Compiled World = admitted desired state
Observed World = live Runtime state
Finding        = desired ↔ observed evidence
Acceptance     = current task/world criteria result
```

Provider 不生产“已验证 World”。Provider 最多生产 Artifact / Provider evidence。

Canonical placement relation 包括：

```text
ON
NEAR
INSIDE + receptacleId
```

这些 relation 必须由 Runtime 执行并重新观察，不能把 Planner declaration 当 truth。

`asset.generate` 是 WorldIR 中的 request-policy boolean；它与已经退役的 `/api/capabilities/asset-generate` 没有关系。

# 7. Finding / Evidence

```text
Finding
├─ code
├─ severity
├─ subject
├─ message
└─ evidence
```

Evidence level 必须保留：

```text
provider_raw_evidence
   ↓
semantic_selected_evidence
   ↓
simulation_validated_evidence
   ↓
agentscape_runtime_verified_evidence
```

低层 evidence 不能向上冒充 Runtime verification。

# 8. Request Identity / Idempotency

所有可能跨进程/网络边界的异步执行必须支持稳定 request identity：

```text
requestId
   ├─ local intent
   ├─ provider execution binding
   └─ retry/recovery lookup
```

重复 submit 不能无条件创建第二个昂贵远程 execution；不确定状态必须显式投影。

# 9. Python SDK Contract

Python SDK 只公开 Connector consumer contract：

```text
ConnectorSession
Capability client
Job client / runner
Artifact transport
normalized request builder / pipeline
```

不再把 Provider-specific client 当稳定 SDK API：

```text
KaggleImageProvider          retired
Modal2DProvider              retired
Modal3DProvider              retired
direct TextTo3DPipeline      retired
```

# 10. Agent Tool Contract

Tool JSON schema、description 与 SkillRegistry definition 是 Agent 参数/结果 contract 的权威。

Prompt policy 只保存跨工具执行不变量，例如：

- mutation 后 fresh replan；
- recovery evidence scope；
- world admission 不得绕过；
- embodied action 使用 Runtime high-level tool。

Prompt 不复制 Provider topology 或完整 Tool schema。


# 11. Developer Observation Contract

Observatory 使用 production Runtime 提供的只读 normalized snapshot；snapshot 不是新的业务 source of truth。

## PhysicsDebugSnapshot

```text
PhysicsDebugSnapshot
├─ schemaVersion
├─ backend
├─ bodies[]
├─ colliders[]
├─ joints[]
├─ contacts[]
├─ nativeGeometry?   optional backend observation
└─ metrics
```

约束：

- 不暴露 solver-private world/body/collider handle；
- collider / contact 必须保留 AgentScape provenance；
- native geometry optional，不能成为跨 backend 必备 contract；
- 若 contact 没有精确 world-space contact point，visualization anchor 必须显式标记其推导类型，不能冒充 solver evidence。

## SpatialDebugSnapshot

```text
SpatialDebugSnapshot
├─ schemaVersion
├─ bounds[]
├─ collisionPairs[]
└─ metrics
```

Ray/support/free-space 的实验 query evidence 可以由 Observatory scenario context 附加，但不得写回 Spatial business truth。

# 12. Contract Versioning

- 只对跨边界稳定 contract 版本化；
- Provider 私有字段自主演化；
- 新版本优先 additive；
- 破坏性协议改动必须新 major/versioned operation；
- 内部 helper/function 不定义协议版本。

# 13. Validation Gates

```text
Connector snapshot
   ↓
Job / Artifact projection
   ↓
Artifact integrity/content verification
   ↓
Asset Compiler / Admission
   ↓
World Compiler / Admission
   ↓
Runtime execution
   ↓
Runtime Verification / Acceptance
```

任何上游 `success` 都不能跳过下游 Gate。
