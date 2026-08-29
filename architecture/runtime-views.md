# Runtime Views

## 1. Agent generation path

```text
User text
  → AgentScape ToolCallingAgent / Skill
  → provider-neutral capability selection
  → Connector session/job request
  → modal-provider
      → optional modal-gen-client
      → modal-2D-client → modal-2D
      → modal-3D-client → modal-3D
  → Artifact descriptor + bytes
  → AgentScape Artifact admission
  → Asset Compiler / Asset admission
  → World Compiler / WorldRuntime
  → Verification
```

Failure ownership：

```text
LLM/VLM decision failure       → AgentScape
Skill/run recovery             → AgentScape
Connector projection failure   → AgentScape connector/job layer
Sidecar restore/cache failure  → modal-provider sidecar
GPU/model failure              → modal-provider provider
Artifact integrity failure     → producer + AgentScape admission gate
Asset/World verification       → AgentScape
```

## 2. Human workflow path

Human 不再通过独立 `modal-inference-hub` 仓库进入系统。

```text
Human
  → AgentScape UI / Task / Run / Editor
  → same provider-neutral capability boundary
  → modal-provider
  → Artifact
  → AgentScape Asset/World pipeline
```

Human 与 Agent 共享 capability contract，但各自的交互状态都由 AgentScape 内部对应 domain 管理。

## 3. 2D → 3D composition

```text
AgentScape intent
  → modal-provider/modal-2D-client
  → modal-provider/modal-2D
  → one or N image artifacts
  → AgentScape/Caller selection
  → modal-provider/modal-3D-client
  → modal-provider/modal-3D
  → GLB artifact
```

Provider-private artifact id、Volume path、Modal call id 不跨越 AgentScape admission boundary。

## 4. 3D input conditioning

```text
source image
  → modal-3D-client public input validation
  → modal-3D conditioning path
       ├─ trustworthy alpha/mask → preserve
       └─ opaque input → segmentation/rembg when required
  → canonical RGBA
  → selected 3D worker
  → GLB
```

Conditioning 是 3D Provider 能力的一部分，不是独立业务仓库。

## 5. EmbodiedGen path

```text
AgentScape/provider request
  → modal-provider/modal-EmbodiedGen runtime
  → pinned upstream EmbodiedGen source
  → provider artifact bundle
  → AgentScape EmbodiedGenAdapter / admission
  → Asset / World pipeline
```

`EmbodiedGen` upstream 不直接成为 AgentScape dependency graph 中的仓库节点。

## 6. Local security gateway

`modal-gen-client` 只是 `modal-provider` 内的可选安全组件：

```text
Browser/WebView
  → modal-gen-client pairing/session/scope
  → provider sidecar
  → provider
```

Native/test caller 可以在满足安全边界时直接调用 sidecar contract；gateway 不是业务 authority。

## 7. Removed runtime paths

以下运行链不再属于目标架构：

```text
AgentScape → kaggle-inference-hub
AgentScape-agent standalone → providers
modal-inference-hub standalone → providers
modal-build standalone → EmbodiedGen runtime
modal-lab → production capability
```
