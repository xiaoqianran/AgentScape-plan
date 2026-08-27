# Runtime Views

# 1. Text → Image → 3D → Asset

```text
Agent / Caller
    │ asset requirement
    ▼
Asset Sourcing
    │ capability: image.generate
    ▼
Capability Catalog
    │ select implementation
    ▼
Provider Adapter
    │
    ▼
modal-2D
    │ provider execution
    ▼
PNG Artifact
    │ verify PNG/digest
    ▼
3D Requirement
    │
    ▼
modal-3D Adapter
    │
    ▼
modal-3D
    │ provider execution
    ▼
GLB Artifact
    │ verify GLB/digest
    ▼
Asset Compiler
    │ semantic / physics / collider / quality passes
    ▼
Asset Admission
    ├─ rejected ──► stop/report
    └─ ready/provisional
             │
             ▼
            Asset
```

失败所有权：

```text
GPU/model failure           → Provider
transport/security failure  → Adapter/Gateway
invalid PNG/GLB             → Artifact Verifier
semantic/physics rejection  → Asset Compiler/Admission
```

# 2. Browser / Local Gateway

```text
Browser / WebView
      │ no privileged credential
      ▼
modal-gen-client
 pairing / origin / scope
      │
      ▼
provider-specific sidecar
      │ privileged local session
      ▼
Provider
```

Gateway 只负责安全与 transport；业务 composition 留在 caller/AgentScape。

# 3. modal-3D-client Local Workflow

```text
Local UI
   │
   ▼
Application API
   ├──────────────┬────────────────┐
   ▼              ▼                ▼
Project       Preprocess      Generation Saga
   │          raw→canonical   intent/idempotency
   │              │          remote/recovery
   └──────────────┼────────────────┘
                  ▼
             Local Artifacts
                  │
                  ▼
              modal-3D
                  │
                  ▼
              GLB Artifact
```

`Project`、`Preprocess`、`Generation Saga` 只通过公开应用边界协作，不直接修改彼此 Store。

# 4. Kaggle Queue Provider

```text
Consumer
   │ submit
   ▼
Consumer API
   │
   ▼
Task Core / Queue
   │ lease
   ├──────────────┐
   ▼              ▼
Worker A       Worker B
   │              │
heartbeat      heartbeat
   │              │
   └──────┬───────┘
          ▼
     upload/complete
          │
          ▼
     Artifact Store
          │
          ▼
     Consumer result
```

Consumer API 永远不暴露 claim/heartbeat/lease；Worker API 不处理 Agent/Asset 产品语义。

# 5. modal-build Build → Runtime

```text
Pinned Upstream
     │
     ▼
Patch Inventory
     │
     ▼
Build Definition
 deps / CUDA / wheels / weights
     │
     ▼
Build Verification
     │
     ▼
Immutable Runtime Artifact
     │
     ▼
Runtime Plane
 ├─ image
 ├─ image3d
 ├─ retexture
 └─ affordance
     │
     ▼
Artifacts / Provider Evidence
```

Runtime request 不允许临时修改 build definition 或在线编译本应离线固定的 CUDA/ABI 依赖。

# 6. World Build

```text
Agent Intent
    │
    ▼
World IR
    │
    ▼
Resolve Assets
    │ 只得到 Asset，不知道 Provider
    ▼
World Compiler
 layout / behavior / physics / relation admission
    │
    ▼
Compiled Desired World
    │
    ▼
WorldRuntime
 instantiate / relations / physics / navigation / interaction
    │
    ▼
Observed World
    │
    ▼
Verification
```

# 7. Desired State Reconciliation

```text
Desired: apple ON table
          │
          ▼
Observed: apple elsewhere
          │
          ▼
Reconcile: place apple
          │
          ▼
Verify: support.on == true
```

```text
Desired: agent HOLDS apple
          │
          ▼
Observed: not held
          │
          ▼
Navigate → pickup
          │
          ▼
Verify: heldBy == agent
```

WorldRuntime 的验证结果不能由 Provider evidence 代替。
