# System Landscape

## 1. 当前系统使命

AgentScape 现在是一个收敛后的系统，而不是由十几个独立仓库拼起来的产品。

```text
User / Human / LLM
        │
        ▼
┌────────────────────────────────────────────┐
│                 AgentScape                 │
│                                            │
│ Agent / LLM / VLM / Skills                │
│ Human UI / Runs / Tasks / Editor           │
│ Connector client / Job projection          │
│ Artifact admission / Asset compiler        │
│ Asset repository / World compiler          │
│ WorldRuntime / Physics / Navigation        │
│ Interaction / Verification                 │
└──────────────────────┬─────────────────────┘
                       │ stable provider contract
                       ▼
┌────────────────────────────────────────────┐
│              modal-provider                │
│                                            │
│ modal-gen-client      local security gw    │
│ modal-2D-client       image sidecar        │
│ modal-2D              image provider       │
│ modal-3D-client       3D sidecar           │
│ modal-3D              3D provider          │
│ modal-EmbodiedGen     build/runtime integ. │
└────────────────────────────────────────────┘
                       │
                       ▼
             Modal / external upstreams
```

`AgentScape-plan` 只记录架构，不在这条运行链上。

## 2. 仓库级 Ownership

### AgentScape

拥有：

- Agent Run、Tool Calling、LLM/VLM gateway；
- Skill 与生成工作流；
- Human-facing editor/task/run UI；
- Connector session/client projection；
- provider-neutral Job/Artifact 语义；
- Asset admission、Asset Compiler、Asset Repository；
- World IR/Compiler/Runtime；
- Physics、Navigation、Interaction、Verification；
- 第一方 Python SDK。

不拥有：Modal 模型部署、Provider 私有 Job、Provider 私有 Artifact 位置、GPU 生命周期。

### modal-provider

拥有：

- Provider 私有执行事实；
- Modal GPU runtime；
- Reference Sidecar 的本地 Job mirror/cache；
- 可选本地安全网关；
- 2D/3D 模型与输入条件处理；
- EmbodiedGen 相关可复现 build artifact 与部署 runtime；
- 上游版本 pin、兼容 patch、provider-level smoke/benchmark。

不拥有：Agent 的业务决策、Asset semantic truth、World truth。

### AgentScape-plan

拥有：Architecture Decision、Migration Ledger、Verified Baseline 文档。

不拥有任何 runtime state。

## 3. `modal-provider` 是 monorepo，不是“旧仓库集合”

当前内部布局：

```text
modal-provider/
├─ modal-gen-client/      optional local security gateway
├─ modal-2D-client/       image Reference Sidecar
├─ modal-2D/              image generation provider
├─ modal-3D-client/       3D Reference Sidecar
├─ modal-3D/              3D generation provider
└─ modal-EmbodiedGen/     EmbodiedGen build/runtime integration
```

这些目录可以有独立的 Python package、lockfile、测试矩阵、Modal app 和发布节奏；但系统文档必须把它们描述为 `modal-provider` 的内部 package/deployment unit。

## 4. 旧仓库映射

| 旧边界 | 当前状态 |
|---|---|
| `AgentScape-agent` | 已并入 `AgentScape` 的 `agent/`、Skills、Gateway/Run 体系 |
| `modal-inference-hub` | 独立仓库退役；Human workflow/UI 归 `AgentScape` |
| `modal-gen-client` | `modal-provider/modal-gen-client` |
| `modal-2D-client` | `modal-provider/modal-2D-client` |
| `modal-2D` | `modal-provider/modal-2D` |
| `modal-3D-client` | `modal-provider/modal-3D-client` |
| `modal-3D` | `modal-provider/modal-3D` |
| `kaggle-inference-hub` | 从目标架构移除 |
| `modal-build` | 能力收敛到 `modal-provider/modal-EmbodiedGen` 及 Provider build/runtime 目录 |
| `EmbodiedGen` | 外部上游依赖；由 Provider 集成层 pin/clone，不作为 AgentScape 独立仓库 |
| `modal-lab` | 独立仓库从目标架构移除；实验随 owning package/repo 保存 |

## 5. Flagship 路径

```text
Text / Human Intent
        │
        ▼
AgentScape Agent or Human UI
        │
        ├─ search existing Asset
        │
        └─ generate missing content
                │
                ▼
        Connector / provider contract
                │
                ▼
           modal-provider
        ┌───────┴────────┐
        │                │
        ▼                ▼
   2D generation     3D generation
        │                │
        └───────┬────────┘
                ▼
        verified Artifact
                │
                ▼
        AgentScape admission
                │
                ▼
         reusable Asset
                │
                ▼
      World Compiler/Runtime
                │
                ▼
           Verification
```

## 6. State Ownership

```text
Agent Run / Skill checkpoint     → AgentScape
Human task/project state         → AgentScape
Connector-facing Job projection  → AgentScape
Provider private execution       → modal-provider component
Provider private artifact bytes  → modal-provider component
Artifact admitted identity       → AgentScape Artifact domain
Asset semantic truth             → AgentScape
World desired/compiled/live      → AgentScape
Build/runtime compatibility      → modal-provider
Architecture decisions           → AgentScape-plan
```

## 7. 全局不变量

1. Provider package 可以拆，仓库边界不因此增加。
2. Provider Artifact 必须通过 AgentScape admission 才能成为 Asset。
3. Provider 不写 World truth。
4. Agent/Human caller 不直接依赖 Provider 私有路径、Volume、Modal call id。
5. `EmbodiedGen` 是 upstream/source dependency，不是 AgentScape runtime authority。
6. 不再为 Kaggle、旧 Hub、旧 Lab 保留平行架构。
