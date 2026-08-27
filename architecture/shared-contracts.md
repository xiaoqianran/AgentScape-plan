# Shared Contracts

本文件只定义跨仓库需要稳定的最小语义。禁止设计覆盖所有 Provider 私有参数的万能 schema。

# 1. CapabilityDescriptor

用途：回答“谁能完成哪一种 Action”。

```text
CapabilityDescriptor
├─ kind              # image.generate / asset3d.generate / ...
├─ provider
├─ operation         # provider-specific operation id
├─ inputSchema
├─ outputs[]
│   ├─ role
│   └─ mediaType
├─ executionMode     # sync / async
├─ cancellable
└─ readiness
```

规则：

- `kind` 使用消费者领域语言。
- `operation` 可以是 Provider 私有 ID。
- Catalog 只做 register/list/find/resolve，不执行 workflow。
- Provider 私有 model/profile/options 保持 Provider namespace。

# 2. ExecutionProjection

用途：消费者观察一次外部执行，不宣称拥有 Provider 内部真值。

```text
ExecutionProjection
├─ requestId
├─ provider
├─ providerExecutionId
├─ state              # queued/running/succeeded/failed/cancelled/uncertain
├─ createdAt
├─ updatedAt
├─ failure?
└─ artifacts[]
```

规则：

- Provider 可以有更丰富内部状态；跨边界只投影稳定子集。
- `uncertain` 是合法状态；submit 超时不能自动等同 `failed`。
- request identity 必须支持幂等绑定。
- 不要求全局统一 Job ID。

# 3. ArtifactDescriptor

```text
ArtifactDescriptor
├─ id
├─ role
├─ mediaType
├─ bytes
├─ digest             # 推荐 sha256:<hex>
├─ producer
│   ├─ provider
│   ├─ operation
│   └─ revision?
└─ lineage[]          # input artifact/request/execution refs
```

规则：

- Content identity 与 location 分离。
- digest 标识的 Artifact 内容不可原地覆盖。
- 派生处理产生新的 Artifact，不修改 source Artifact 的事实身份。
- 跨仓优先传 Descriptor + bytes/location，不传私有 ORM/Job object。

# 4. Finding

```text
Finding
├─ code
├─ severity           # info/advisory/error
├─ subject
├─ message
└─ evidence
```

Evidence 层级必须保留，禁止向上冒充：

```text
provider_raw_evidence
   ↓
semantic_selected_evidence
   ↓
simulation_validated_evidence
   ↓
agentscape_runtime_verified_evidence
```

例如 raw grasp、semantic selected grasp、SAPIEN validated grasp、AgentScape/Rapier verified grasp 必须是不同 evidence level。

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
├─ quality/admission
└─ sourceArtifactLineage
```

只有 Asset Compiler / Admission 可以决定：

```text
ready
provisional
rejected
```

Provider 不得直接声明 AgentScape Asset 已验证。

# 6. World

```text
World IR       = desired intent
Compiled World = admitted desired state
Observed World = live Runtime state
Finding        = desired ↔ observed difference/evidence
```

Provider 不生产“已验证 World”。Provider 最多生产 Artifact / Evidence。

# 7. Request Identity / Idempotency

所有可能跨进程/网络边界的异步执行必须支持稳定 request identity。

```text
requestId
   │
   ├─ local intent
   ├─ provider execution binding
   └─ retry/recovery lookup
```

规则：

- 同一 request identity 的重复 submit 不能无条件创建第二个昂贵远程执行。
- 出现“远程可能创建成功、本地未绑定”的情况时进入 `uncertain`，不得自动重复计费。
- Sidecar/Gateway 可以保存投影，但 Provider execution identity 仍由 Provider 拥有。

# 8. Contract Versioning

- 只对跨仓稳定 Contract 版本化。
- Provider 私有字段放 Provider namespace，自主演化。
- 新版本优先 additive。
- 破坏性改动必须新 major/versioned operation。
- 内部 helper/function 不定义协议版本。

# 9. Validation Gates

```text
Provider output
   ↓
Artifact structural/content verification
   ↓
Asset Compiler / Admission
   ↓
World Compiler / Admission
   ↓
Runtime execution
   ↓
Runtime verification
```

任何上游 `success` 都不能跳过下游 Gate。
