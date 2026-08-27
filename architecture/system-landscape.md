# System Landscape

## 1. 系统使命

AgentScape 多仓库系统不是一个“大客户端”或“万能 Generation Orchestrator”。它由四个稳定层组成：

```text
Caller       决定“做什么、为什么、选哪个结果”
Capability   执行“生成/推理/转换”
Asset        决定“这个内容是什么、是否可复用”
World        决定“它在哪里、和谁有什么关系、如何交互”
```

最重要的系统目标是：

> **Text / Human Intent → Caller Workflow → Verified Artifact → Reusable Asset → Compiled & Verified World。**

## 2. 系统总图

```text
                              User Intent
                                  │
                 ┌────────────────┴────────────────┐
                 │                                 │
                 ▼                                 ▼
       ┌──────────────────┐             ┌──────────────────────┐
       │ AgentScape-agent │             │ modal-inference-hub  │
       │                  │             │                      │
       │ LLM/VLM          │             │ Human UI / Project   │
       │ Agent Run        │             │ manual selection     │
       │ Skills/Tools     │             │ workflow history     │
       └────────┬─────────┘             └──────────┬───────────┘
                │                                  │
                │              CALLER LAYER        │
                └───────────────┬──────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ modal-2D-client  │  │ modal-3D-client  │  │ other adapters   │
│ Reference Sidecar│  │ Reference Sidecar│  │ Kaggle/Embodied  │
└─────────┬────────┘  └─────────┬────────┘  └─────────┬────────┘
          │                     │                     │
          ▼                     ▼                     ▼
     ┌─────────┐           ┌─────────┐         ┌──────────────┐
     │modal-2D │           │modal-3D │         │ Providers    │
     │image gen│           │3D gen   │         │              │
     └────┬────┘           └────┬────┘         └──────┬───────┘
          │                     │                     │
          └───────────────┬─────┴─────────────────────┘
                          │
                          │ Artifact
                          ▼
               ┌─────────────────────────┐
               │       AgentScape        │
               │                         │
               │       ASSET LAYER       │
               │ Artifact Admission      │
               │ Asset Compiler          │
               │ Asset Repository        │
               │ Asset Search            │
               │                         │
               │       WORLD LAYER       │
               │ World IR                │
               │ World Compiler          │
               │ WorldRuntime            │
               │ Physics/Nav/Interaction │
               │ Verification            │
               └─────────────────────────┘
```

`AgentScape-client` 是 AgentScape Core 的 Reference SDK/CLI；测试程序也可以直接调用任意公开边界。`modal-gen-client` 只在 Browser/WebView 需要本机特权隔离时作为可选 Security Gateway，不是业务编排层。

## 3. 旗舰 Agent 路径

未来最重要的系统 E2E：

```text
User Text
   │
   ▼
AgentScape-agent
   │
   ▼
理解 World / Asset Intent
   │
   ├─ search existing asset
   │
   └─ missing
        │
        ▼
   source_3d_asset Skill
        │
        ├─ generate N image candidates ──► modal-2D-client ──► modal-2D
        │
        ├─ VLM evaluate / rank / retry
        │
        ├─ selected image
        │
        ├─ generate 3D ──────────────────► modal-3D-client ──► modal-3D
        │
        ├─ Artifact verify
        │
        └─ publish_asset ─────────────────► AgentScape
                                                  │
                                                  ▼
                                           Asset Repository
                                                  │
                                         World Intent / Relations
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

因此“Text→3D”默认是 **Caller Skill**：Text → 多张 2D 候选 → 选择 → Image→3D。只有未来某个 Provider 真正原生支持 text input 时，才新增 native text-to-3D capability。

## 4. Human 与 Agent 是平级 Caller

```text
Candidate Images
      │
      ├─────────────────────┐
      ▼                     ▼
modal-inference-hub   AgentScape-agent
Human selection       VLM/Agent selection
      │                     │
      └──────────┬──────────┘
                 ▼
          selected image/mask
                 │
                 ▼
          modal-3D-client
```

任何一个 Caller 都不能成为 Provider 的唯一上级。

## 5. Semantic Preprocess 与 Model Input Conditioning

两者必须分离：

```text
Caller semantic preprocess
“用户/Agent 想要哪个物体？”

modal-inference-hub: human mask/component selection
AgentScape-agent:    VLM/object selection
                 │
                 │ image + optional trustworthy mask
                 ▼
modal-3D-client
transport / durable execution only
                 │
                 ▼
modal-3D InputConditioner
“这个模型怎样稳定消费输入？”
                 │
        ┌────────┴────────┐
        │                 │
valid alpha/mask?       none
        │                 │
     preserve       auto segmentation/rembg
        │                 │
        └────────┬────────┘
                 ▼
       crop/center/normalize
                 │
                 ▼
       model-private canonical input
```

`rembg` 是 `InputConditioner` 的一种策略，不是系统级独立业务层。

## 6. Asset 与 World 分离

```text
GLB Artifact
    │
    ▼
Asset Compiler
    │
    ▼
Asset Repository
    │
    │ reusable identity
    ▼
Asset: red_apple

World Instance
    │ references Asset
    ├─ instance: apple_01
    ├─ relation: ON table_01
    └─ transform: compiled by World Compiler
```

Asset 不保存某一次 World Placement；位置/关系属于 World Instance。

## 7. 仓库角色

| Repository | 稳定角色 | 是否业务真值 Owner |
|---|---|---|
| `AgentScape` | Asset + World Domain Core | Asset / World |
| `AgentScape-agent` | Agentic Orchestration Caller | Agent Run / Skill state |
| `AgentScape-client` | AgentScape Reference SDK/CLI | 否 |
| `modal-inference-hub` | Human Workflow Caller | Project / human workflow |
| `modal-gen-client` | Optional Local Security Gateway | security session only |
| `modal-2D-client` | Image Provider Reference Sidecar | local execution mirror/cache |
| `modal-2D` | Image Generation Provider | model/runtime execution |
| `modal-3D-client` | 3D Provider Reference Sidecar | local execution mirror/cache |
| `modal-3D` | 3D Provider + Input Conditioning | model/runtime execution |
| `kaggle-inference-hub` | Queue-backed Provider | task/lease/worker |
| `modal-build` | Reproducible Build + Embodied Runtime | build/runtime identity |
| `EmbodiedGen` | Pinned Readonly Upstream | upstream only |
| `modal-lab` | Research / Verification Incubator | experiment evidence |
| `AgentScape-plan` | Architecture Authority | architecture decisions |

**Submodule ≠ runtime dependency。** 一个仓库可以只是 pinned source、integration fixture、E2E target 或 development workspace。

## 8. 全局 State Ownership

```text
Agent Run / Skill checkpoint    → AgentScape-agent
Human Project / selection       → modal-inference-hub
Provider execution              → Provider / Reference Sidecar projection
Gateway pairing/scope           → modal-gen-client
Artifact content identity       → Producer + digest
Artifact verification           → Verifier Finding
Asset semantic truth            → AgentScape
World desired/compiled state     → AgentScape World Domain
Live observed world             → AgentScape WorldRuntime
Research result                 → modal-lab
Build identity                  → modal-build
```

## 9. 全局不变量

### INV-01 — Caller 不拥有 Provider 私有真值

Agent/Hub 可以持有 job reference，但不能共享 Provider 私有 Store/ORM/model lifecycle。

### INV-02 — 成功不能折叠

```text
Execution succeeded
   ≠ Artifact valid
   ≠ Asset admitted
   ≠ World ready
   ≠ Interaction verified
```

### INV-03 — Job ID 不全局统一

Agent Run、Sidecar Job、Provider task、Modal FunctionCall 可以有不同 ID，通过 request identity、lineage 与 Artifact digest 关联。

### INV-04 — Artifact 是跨生成边界的主要事实

优先传 `ArtifactDescriptor + bytes/location`，不传私有执行对象。

### INV-05 — Agent 选 Skill，Skill 组合 Tool

Agent 不应该逐次 LLM 决策 `poll/download/hash`。长确定性步骤由 Skill workflow 执行，Agent 只在需要判断/重规划的位置介入。

### INV-06 — UI/Client 不拥有 Asset/World 真值

Browser、Tauri、CLI、Hub UI、Reference Sidecar 都不能宣布一个 Provider output 已经是 ready Asset/World。

### INV-07 — Research 不是 Production Dependency

`Runtime → modal-lab` 永久禁止。实验必须通过 Promotion Gate 进入 production repo。

## 10. Research References

- Terraform Core / Provider: https://developer.hashicorp.com/terraform/plugin/how-terraform-works
- Bazel artifact/action model: https://bazel.build/concepts/build-ref
- OCI Image Spec: https://github.com/opencontainers/image-spec
- MLIR Pass Management: https://mlir.llvm.org/docs/PassManagement/
- Kubernetes Controllers: https://kubernetes.io/docs/concepts/architecture/controller/
- NVIDIA Triton: https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/architecture.html
- KServe V2 Protocol: https://kserve.github.io/website/docs/concepts/architecture/data-plane/v2-protocol
- Celery: https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- Temporal: https://docs.temporal.io/
- Nix: https://nix.dev/manual/nix/latest/language/derivations
- MLflow Tracking: https://mlflow.org/docs/latest/ml/tracking/
- Tauri Capabilities: https://v2.tauri.app/security/capabilities/
- C4 Model: https://c4model.com/
- arc42: https://arc42.org/
- LangGraph concepts: https://langchain-ai.github.io/langgraph/
- OpenUSD composition: https://openusd.org/release/intro.html
