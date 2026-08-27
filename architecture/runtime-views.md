# Runtime Views

# 1. Text → Agent → 3D Asset → World

这是未来 AgentScape 的旗舰自动化路径。

```text
User Text
   │
   ▼
AgentScape-agent
   │ parse intent
   ▼
source_3d_asset Skill
   │
   ├─ search existing Asset ───────────────► AgentScape
   │        │
   │        └─ hit → reuse
   │
   └─ miss
        │
        ▼
   generate N images
        │
        ▼
   modal-2D-client
        │
        ▼
     modal-2D
        │
        ▼
   Image Artifacts
        │
        ▼
   VLM evaluate / rank / retry
        │
        ▼
   selected image + optional semantic mask
        │
        ▼
   modal-3D-client
        │
        ▼
     modal-3D
        │ InputConditioner
        ▼
      3D Worker
        │
        ▼
     GLB Artifact
        │
        ▼
   Artifact verification
        │
        ▼
   publish_asset
        │
        ▼
     AgentScape
   Asset Compiler
        │
        ▼
  Asset Repository
        │
        ▼
    World Intent
        │
        ▼
   World Compiler
        │
        ▼
   WorldRuntime
        │
        ▼
   Verification
```

失败所有权：

```text
LLM/VLM decision failure          → AgentScape-agent
Provider transport/recovery       → Reference Sidecar
GPU/model failure                 → Provider
invalid PNG/GLB                   → Artifact verifier
Asset semantic/admission failure  → AgentScape Asset Domain
World relation/physics failure    → AgentScape World Domain/Runtime
```

# 2. Human modal-inference-hub Workflow

```text
Human Prompt / Source Image
        │
        ▼
modal-inference-hub Project
        │
        ├─ generate image candidates ─► modal-2D-client
        │
        ├─ preview / compare
        │
        ├─ manual select
        │
        ├─ optional semantic mask/component edit
        │
        └─ generate 3D ───────────────► modal-3D-client
                                              │
                                              ▼
                                           modal-3D
                                              │
                                              ▼
                                          GLB Artifact
                                              │
                    ┌─────────────────────────┴─────────────┐
                    ▼                                       ▼
              keep in Project                     publish to AgentScape
```

Human Hub 与 AgentScape-agent 是平级 Caller；二者共享 Sidecar/Provider 边界，但不共享 Project/AgentRun state。

# 3. Reference Sidecar Pattern

2D 与 3D Client 必须保持相似职责：

```text
Caller
  │ request
  ▼
Reference Sidecar
  │
  ├─ validate public request
  ├─ stable request identity
  ├─ local durable execution projection
  ├─ provider submit/poll/cancel
  ├─ restart recovery
  ├─ artifact transport
  ├─ content verification
  └─ local content-addressed cache
  │
  ▼
Provider
```

Sidecar 不做：Project、Agent planning、Asset compilation、World placement。

# 4. modal-3D Input Conditioning

目标 public contract 不应该要求 Caller 理解模型私有 canonical 规则。

```text
Caller image
+ optional trusted mask/alpha
          │
          ▼
modal-3D-client
transport unchanged
          │
          ▼
modal-3D Provider API
          │
          ▼
InputConditioner
   │
   ├─ decode / EXIF / color
   │
   ├─ valid alpha/mask?
   │      ├─ yes → preserve
   │      └─ no  → segmentation/rembg
   │
   ├─ foreground bbox
   ├─ crop / center / scale
   └─ model-private canonical representation
          │
          ▼
Model Adapter / Worker
```

当前部署 baseline 仍要求：

```text
1024×1024 RGBA PNG
alpha channel required
role = canonical_rgba
```

`041-modal-3d-provider` 记录该旧 contract 的真实可用性；未来迁移 InputConditioner 时必须保持 4 模型 E2E parity。

# 5. Candidate Selection Ownership

```text
Image Candidates
      │
      ├─────────────────────┐
      ▼                     ▼
modal-inference-hub   AgentScape-agent
Human judgment        VLM/Agent judgment
      │                     │
      └──────────┬──────────┘
                 ▼
        selected image/mask
```

Provider/Sidecar 永远不决定“哪个候选符合用户意图”。

# 6. Artifact → Asset Publication

```text
GLB Artifact
   │ descriptor + bytes/location
   ▼
AgentScape Artifact Admission
   │ structural/content findings
   ▼
Asset Compiler
   │ semantic / geometry / collider / physics / actions
   ▼
Asset Admission
   ├─ rejected
   ├─ provisional
   └─ ready
        │
        ▼
Asset Repository
```

Provider `success` 不能跳过 AgentScape Asset Admission。

# 7. Asset → World Placement

```text
Reusable Asset: red_apple
          │
          │ reference
          ▼
World Instance: apple_01
          │
          ├─ ON table_01
          ├─ NEAR cup_01
          └─ transform = compiler output
          │
          ▼
World Compiler
          │
          ▼
Desired World
          │
          ▼
WorldRuntime
          │
          ▼
Observed World
          │
          ▼
Verification
```

Asset identity 与 World placement 永久分离。

# 8. Desired State Reconciliation

```text
Desired: apple ON table
          │
          ▼
Observed: apple elsewhere
          │
          ▼
Reconcile / placement
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

# 9. Browser / Local Security Gateway

```text
Browser / WebView
      │ no privileged credential
      ▼
modal-gen-client
 pairing / origin / scope
      │
      ▼
Reference Sidecar
      │
      ▼
Provider
```

这是部署 Adapter，不是业务 workflow。Server-side Agent/Hub 可以直接调用 Sidecar。

# 10. Kaggle Queue Provider

```text
Caller
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
   └──────┬───────┘
          ▼
     upload/complete
          │
          ▼
     Artifact Store
```

Consumer API 永远不暴露 claim/heartbeat/lease。

# 11. modal-build Build → Runtime

```text
Pinned Upstream
     │
     ▼
Patch Inventory
     │
     ▼
Build Definition
     │
     ▼
Build Verification
     │
     ▼
Immutable Runtime Artifact
     │
     ▼
Embodied Runtime Plane
     │
     ▼
Artifacts / Provider Evidence
```

Runtime request 不在线重建本应离线固定的 CUDA/ABI dependency。
