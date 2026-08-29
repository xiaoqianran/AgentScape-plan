# System Landscape

## 1. 当前系统使命

AgentScape 是一个收敛后的 embodied-agent product/runtime monorepo；远程生成执行由兄弟仓库 `modal-provider` 提供。`AgentScape-plan` 只记录架构，不参与运行时。

```text
User / Human / LLM
        │
        ▼
┌──────────────────────────────────────────────┐
│                  AgentScape                  │
│                                              │
│ Studio / Observatory / Agent / Skills        │
│ GenerationRuntime                            │
│ Connector session + capability snapshot      │
│ Job / Artifact projection                    │
│ Asset publication / Compiler / Admission     │
│ WorldIR / Compiler / Runtime / Verification  │
│ Python SDK (Connector consumer)               │
└──────────────────────┬───────────────────────┘
                       │ stable Connector/provider contract
                       ▼
┌──────────────────────────────────────────────┐
│               modal-provider                 │
│                                              │
│ modal-gen-client      local security gateway │
│ modal-2D-client       image sidecar          │
│ modal-2D              image provider         │
│ modal-3D-client       3D sidecar             │
│ modal-3D              3D provider            │
│ modal-EmbodiedGen     build/runtime integ.   │
└──────────────────────────────────────────────┘
                       │
                       ▼
              Modal / external upstreams
```

## 2. AgentScape Ownership

AgentScape owns:

- Agent Run、Tool Calling、LLM/VLM gateway；
- domain skill packs 与可组合 Agent prompt policies；
- Human-facing Studio / Task / Run / Editor；
- Developer-facing Observatory（Physics/Spatial observation surface）；
- `GenerationRuntime` provider-neutral generation composition；
- Connector session、capability snapshot 与 ProviderRegistry projection；
- Job / Artifact projection、Artifact integrity gate；
- Artifact → Asset publication；
- Asset Compiler / Asset admission / reusable Asset truth；
- WorldIR、World Compiler、WorldRuntime；
- Physics、Navigation、Interaction、Verification / Acceptance；
- 第一方 Python SDK；
- 同仓 `api/`、`services/`、tests、tooling。

AgentScape does not own:

- remote Provider implementation；
- Provider-private credential；
- GPU/model lifecycle；
- Provider-private Job/Artifact storage；
- Modal call id / private Volume path。

## 3. Generation control plane

AgentScape 只有一条正式远程生成主链：

```text
Studio / Agent / World retry
        │
        ▼
GenerationRuntime
        │
        ▼
Connector session
        │
        ▼
capability snapshot
        │
        ▼
ProviderRegistry projection
        │
        ├─ Job submit/reconcile/cancel
        └─ Artifact descriptor/bytes
                │
                ▼
          Asset publication
                │
                ▼
       Compiler / Asset admission
```

不再存在：

```text
LegacyAuthoringShell
runtime.authoring
direct HTTP asset generator
/api/capabilities/asset-generate
source-coded remote Provider placeholders
```

`ProviderRegistry` 默认不认识任何远程 Provider id。远程 Provider 只能由当前 Connector capability snapshot 动态进入 registry；snapshot 清除后对应 Provider 也必须消失。

## 4. Caller ownership

Human 与 Agent 都在 AgentScape 内部，但保持不同 caller state：

```text
Human → Studio UI → GenerationRuntime / World tools
Agent → Agent skills → GenerationRuntime / World tools
```

两者共享 provider-neutral capability / Job / Artifact / Asset / World contract，不共享 UI state 或 Agent Run state。

`GenerationJobCenter` 是 Studio UI，归 `studio/ui/generation/`；Generation domain 不拥有 DOM/UI。


## 5. Developer Observatory

Observatory 是 AgentScape 内的 Developer Product Surface，不是新的 domain truth owner：

```text
Production Runtime / Domain
       ▲             ▲
       │             │
    Studio      Observatory
     uses        observes

Production ─────X────► Observatory
```

规则：

- Observatory 可以 import production Physics/Spatial/Asset/World contract；
- production module 禁止反向依赖 Observatory；
- Scenario 必须驱动 production implementation；
- synthetic geometry 可以存在，但 production Manifest/Schema 不得复制；
- UI 只消费 normalized debug snapshot，不穿透 solver-private world handle。

当前正式 Lab：Physics、Spatial。未来 Navigation/Interaction/Agent Lab 继续遵循同一规则。

## 6. Python SDK

Python SDK 是 Unified Connector consumer，而不是 Provider client collection。

公开目标 surface：

```text
ConnectorSession
ConnectorCapabilityClient
ConnectorJobClient / Runner
ConnectorArtifactTransport
ConnectorTextTo3DPipeline
normalized request builders / contracts
```

Kaggle/direct Modal Provider client 与 direct pipeline 已从 public surface 删除。

## 7. `modal-provider` monorepo

```text
modal-provider/
├─ modal-gen-client/
├─ modal-2D-client/
├─ modal-2D/
├─ modal-3D-client/
├─ modal-3D/
└─ modal-EmbodiedGen/
```

这些是 package/deployment/runtime boundary，不是新的 repository-level architecture authority。

## 8. State Ownership

```text
Agent Run / mutation identity       → AgentScape
Human Studio state                 → AgentScape
Connector session                  → AgentScape consumer projection
Capability snapshot projection     → AgentScape
Generation Job projection          → AgentScape
Provider private execution         → modal-provider
Provider private artifact storage  → modal-provider
Admitted Artifact identity         → AgentScape
Asset semantic truth               → AgentScape
World desired/compiled/live truth  → AgentScape
Runtime verification/acceptance    → AgentScape
Observatory debug projection        → AgentScape developer surface
Build/runtime compatibility        → modal-provider
Architecture decisions             → AgentScape-plan
```

## 9. Global invariants

1. Provider package 可以拆，repository boundary 不因此增加。
2. AgentScape source code 不硬编码远程 Provider topology。
3. Provider output 必须先通过 Artifact/Asset admission，不能直接成为 Asset truth。
4. Provider 不写 World truth。
5. Generation domain 不依赖 Studio；Studio 可以消费 Generation domain。
6. World core 不依赖 Generation/Studio/Agent，只消费窄 Asset contract。
7. `EmbodiedGen` 是 upstream/source dependency，不是 AgentScape runtime authority。
8. Kaggle、旧 Hub、旧 Lab、LegacyAuthoring/direct generator 不再拥有平行生产主链。
9. Production runtime never depends on Observatory.
