
> Replaceable Physics/Navigation backend ownership、native leakage 与 parity/conformance 规则详见 [`runtime-backend-plane.md`](./runtime-backend-plane.md) 与 ADR-0010。
# Runtime Views

## 1. Agent generation path

```text
User text
  → ToolCallingAgent
  → domain skill pack
  → GenerationRuntime
  → Connector session
  → capability snapshot / provider projection
  → Job submit / reconcile
  → Artifact descriptor + bytes
  → Artifact integrity gate
  → Asset publication
  → Asset Compiler / admission
  → WorldIR / WorldRuntime
  → Verification / Acceptance
```

Failure ownership：

```text
LLM/VLM decision failure        → AgentScape Agent
Skill/run recovery              → AgentScape Agent
Connector/session failure       → AgentScape Generation consumer layer
Provider execution failure      → modal-provider
Artifact integrity failure      → producer + AgentScape admission
Asset/World verification        → AgentScape
```

AgentScape 不在源码中预先知道远程 Provider id；选择空间来自当前 Connector capability snapshot。

## 2. Human generation path

```text
Human
  → Studio
  → studio/ui/generation/GenerationJobCenter
  → GenerationRuntime
  → same Connector / Job / Artifact boundary
  → Asset / World
```

`GenerationJobCenter` 只是 Human UI，不拥有 Job truth 或 Provider execution。

## 3. Missing Asset in canonical World pipeline

```text
WorldIR
  → resolve reusable Asset
  → missing
  → generation policy allows retry
  → GenerationRuntime.resolveAssetRequest
  → Connector capability / Job / Artifact
  → Asset publication/admission
  → retry World compilation once
  → final world admission
```

`asset.generate` 是 WorldIR 的 generation-policy boolean，不是旧 `/api/capabilities/asset-generate` deployment endpoint。

## 4. Provider snapshot lifecycle

```text
paired Connector session
  → capability snapshot
  → normalize
  → apply to ProviderRegistry
  → Connector owns projected Provider ids

new snapshot
  → replace Connector-owned projection

session revoked / snapshot cleared
  → remove Connector-owned Provider ids
  → do not restore source-coded placeholders
```

显式 local/test provider ownership 不得被 Connector snapshot 覆盖。

## 5. Artifact → Asset

```text
Provider-private result
  → Connector Artifact projection
  → descriptor / bytes / digest
  → local structural/content verification
  → Artifact Registry / byte store
  → Asset publication
  → Asset Compiler
  → ready / provisional / rejected
```

Provider success 不等于 Asset ready。

## 6. Generated World

```text
Planner
  → proposeWorldIR
  → Runtime-issued revision/provenance
  → runWorldPipeline
  → asset resolution/generation
  → deterministic layout
  → behavior/physics/relation admission
  → instantiate
  → validation / repair
  → acceptance
  → world-ready | world-provisional | world-rejected
```

`ON / NEAR / INSIDE` 都必须由 Runtime 物理执行/重新验证，Planner declaration 不等于 World truth。

## 7. Python SDK

```text
Python caller
  → ConnectorSession
  → Capability client
  → Job runner/client
  → Artifact transport
  → normalized pipeline/request builder
```

Python SDK 不再有 direct Kaggle/Modal Provider client，也不再提供 `reconstruct-direct`。

## 8. Agent prompt / skill runtime

```text
ToolCallingAgent
  → build prompt from policy modules
  → attach current structured Tool definitions
  → LLM call
  → SkillRegistry dispatch
  → domain skill pack handler
  → Runtime mutation/verification
  → fresh replan after mutation
```

Tool schema/description 是参数与结果 contract 的权威。Prompt policy 只保存跨工具执行不变量，不能复制一份 Provider/World schema 当作隐形架构。

## 9. 2D → 3D provider composition

Provider 内部仍可组合 2D/3D package：

```text
Connector-normalized intent
  → modal-provider 2D execution
  → image Artifact
  → modal-provider 3D execution
  → GLB Artifact
  → AgentScape admission
```

Provider-private Artifact id、Volume path、Modal call id 不跨越 AgentScape admission boundary。

## 10. EmbodiedGen

```text
Connector request
  → modal-provider/modal-EmbodiedGen
  → pinned upstream source/runtime
  → Artifact bundle
  → AgentScape Artifact/Asset admission
```

AgentScape 可保留 Asset compatibility adapter 来理解既有上游 payload，但 Provider-specific import tool 不属于默认 Agent capability surface。


## 11. Observatory observation path

```text
Developer
  → /observatory/
  → ScenarioRunner + fixed-step SimulationClock
  → production PhysicsSystem / SpatialSystem
  → normalized debugSnapshot
  → visualizer / assertion / backend comparison
```

Physics backend comparison：

```text
same Scenario + same fixed dt
   ├─ Rapier production backend
   └─ Jolt production backend
        ↓
normalized body/collider/joint/contact snapshot
        ↓
comparison
```

Observatory 不直接读取 solver-private world handle；native debug geometry 只能通过 backend optional observation contract 暴露。

Spatial/BVH：

```text
WorldRuntime
  → ThreeBvhRuntime
  → ensureBoundsTrees on world admission/spawn
  → SpatialSystem raycast
  → Observatory only observes results
```

## 12. Removed runtime paths

```text
runtime.authoring
LegacyAuthoringShell
AgentScape → direct HTTP asset generator
AgentScape ProviderRegistry → built-in remote placeholders
Python SDK → direct Kaggle/Modal Provider
AgentScape → kaggle-inference-hub
modal-inference-hub standalone → providers
modal-build standalone → EmbodiedGen runtime
```
