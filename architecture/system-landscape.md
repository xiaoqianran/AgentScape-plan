# System Landscape

## 1. 系统使命

AgentScape 不是“管理所有 Client / Connector / GPU / Job / Project 的超级应用”。

它的核心使命是：

> **通过清晰的 Port 调用独立执行能力，把可验证 Artifact 编译成 Asset，再把 Asset 与 World IR 编译成可验证的 Live World。**

## 2. 系统总图

```text
                              User
                               │
                               ▼
┌────────────────────────── AgentScape ──────────────────────────┐
│                                                               │
│  Agent → Skills                                               │
│           │                                                   │
│           ├──────────────┐                                    │
│           ▼              ▼                                    │
│     Asset Sourcing    World Intent                            │
│           │              │                                    │
│           ▼              ▼                                    │
│       Artifact         World IR                               │
│           │              │                                    │
│           ▼              ▼                                    │
│     Asset Compiler   World Compiler                           │
│           │              │                                    │
│           └────── Asset ─┘                                    │
│                    │                                          │
│                    ▼                                          │
│               WorldRuntime                                    │
│          Physics / Nav / Interaction                          │
│                    │                                          │
│                    ▼                                          │
│               Verification                                    │
└──────────────┬───────────────────────────────┬────────────────┘
               │ Port                          │ Artifact/Finding
               ▼                               ▼
       ┌──────────────┐                 ┌──────────────┐
       │ Local Gateway│                 │Artifact Store│
       └──────┬───────┘                 └──────────────┘
              │
      ┌───────┼──────────────┬───────────────────┐
      ▼       ▼              ▼                   ▼
  modal-2D  modal-3D   Kaggle Provider      Embodied Runtime
  Provider  Provider

  modal-lab ──research only──► Productionization Gate
  EmbodiedGen ──pinned upstream──► modal-build
```

注：这是**逻辑边界**，不是强制网络拓扑。Server 可以直连 Provider；Browser/桌面场景可以经 Local Security Gateway。

## 3. 仓库角色

| Repository | 稳定角色 | 生产运行时是否必需 |
|---|---|---:|
| `AgentScape` | Domain Core / Aggregator | 是 |
| `AgentScape-client` | Reference SDK / CLI / Harness | 否 |
| `modal-gen-client` | Local Security Gateway | 按部署模式 |
| `modal-2D-client` | modal-2D Reference Sidecar | 否 |
| `modal-2D` | Image Generation Provider | 按需 |
| `modal-3D-client` | Local 3D Workflow Host | 否 |
| `modal-3D` | Multi-model 3D Provider | 按需 |
| `kaggle-inference-hub` | Queue-backed Execution Provider | 按需 |
| `modal-build` | Reproducible Build + Embodied Runtime | 按需 |
| `EmbodiedGen` | Pinned Readonly Upstream | 否 |
| `modal-lab` | Research Incubator | 否 |
| `AgentScape-plan` | Architecture Authority | 否 |

**Submodule ≠ Runtime dependency。** Submodule 可以只是 pinned source、E2E target、integration fixture 或 development workspace。

## 4. 全局 State Ownership

```text
Provider Execution        → Provider / Provider-side execution host
Gateway Session/Scope     → Local Gateway
Artifact content          → immutable bytes + digest
Artifact validation       → Verifier Finding
Asset semantic truth      → AgentScape Asset Compiler
Compiled desired world    → AgentScape World Compiler
Live observed world       → AgentScape WorldRuntime
Agent decision trace      → Agent
Research result           → modal-lab experiment evidence
Build identity            → modal-build Build Plane
```

## 5. 不变量

### INV-01 — 成功不能折叠

```text
Execution succeeded
   ≠ Artifact valid
   ≠ Asset admitted
   ≠ World ready
   ≠ Interaction verified
```

### INV-02 — Job ID 不全局统一

允许 consumer execution id、provider job id、Modal FunctionCall id、Kaggle task id 同时存在。通过 request identity、lineage、artifact digest 关联。

### INV-03 — Artifact 是主要跨仓库数据边界

优先传 `ArtifactDescriptor + bytes/location`，不传 Provider 私有 ORM/Store/Job object。

### INV-04 — Capability 只描述“谁能做什么”

```text
Capability = 能不能做
Execution  = 这次怎么做
Artifact   = 做出了什么
Asset      = 内容在 AgentScape 中意味着什么
```

### INV-05 — Provider 私有实现不得进入 AgentScape Domain

AgentScape Core 不认识 Modal credential、Kaggle lease、HF token、CUDA package、FastSAM cache、Provider 私有数据库。

### INV-06 — UI/Client 不拥有领域真值

Browser、Tauri、CLI、Debug UI、Reference Client 都是 Adapter/View/Harness。

### INV-07 — Research 不是 Production Dependency

`AgentScape Runtime -> modal-lab` 永久禁止。实验必须通过 productionization gate。

## 6. 三个平面

```text
┌──────────────── Control Plane ────────────────┐
│ select / configure / authorize / recover     │
└──────────────────────┬────────────────────────┘
                       ▼
┌──────────────── Data Plane ───────────────────┐
│ generate / transform / compile / interact    │
└──────────────────────┬────────────────────────┘
                       ▼
┌──────────────── Evidence Plane ───────────────┐
│ observe / verify / explain / trace            │
└───────────────────────────────────────────────┘
```

Evidence Plane 只投影已有事实，不成为新的业务 State Owner。

## 7. Research References

这些来源只提供架构原则，不成为运行时依赖：

- Terraform Core / Provider: https://developer.hashicorp.com/terraform/plugin/how-terraform-works
- Bazel artifact/action model: https://bazel.build/concepts/build-ref
- OCI Image Spec / Descriptor: https://github.com/opencontainers/image-spec
- MLIR Pass Management: https://mlir.llvm.org/docs/PassManagement/
- Kubernetes Controllers: https://kubernetes.io/docs/concepts/architecture/controller/
- NVIDIA Triton Architecture: https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/architecture.html
- KServe V2 Inference Protocol: https://kserve.github.io/website/docs/concepts/architecture/data-plane/v2-protocol
- Celery task queue: https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- Temporal durable execution: https://docs.temporal.io/
- Nix derivations/reproducible builds: https://nix.dev/manual/nix/latest/language/derivations
- MLflow Tracking: https://mlflow.org/docs/latest/ml/tracking/
- Tauri Capabilities: https://v2.tauri.app/security/capabilities/
- C4 Model: https://c4model.com/
- arc42: https://arc42.org/
