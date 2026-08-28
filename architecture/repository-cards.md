# Repository Architecture Cards

本文件是仓库级目标架构权威。图中的“模块”表示职责边界，**不等于必须一个模块一个文件**；能在一个文件内高内聚实现时优先保持简单。

---

# CARD 01 — AgentScape

**定位：Asset + World Domain Core / 模块化单体。**

```text
                 External Caller
   AgentScape-agent / Hub / SDK / Tests
                         │
                         ▼
┌──────────────────── AgentScape ─────────────────────┐
│                                                    │
│ [1] Domain API                                     │
│ ┌────────────────────────────────────────────────┐ │
│ │ Asset API                                      │ │
│ │ World API                                      │ │
│ │ Runtime/Interaction API                        │ │
│ └──────────────┬─────────────────────┬───────────┘ │
│                │                     │             │
│                ▼                     ▼             │
│ [2A] Asset Domain              [2B] World Domain   │
│ ┌───────────────────────┐     ┌──────────────────┐ │
│ │ Artifact Admission    │     │ World IR         │ │
│ │ Asset Compiler        │     │ Relations        │ │
│ │ Asset Admission       │     │ Layout Intent    │ │
│ │ Asset Repository      │     │ World Compiler   │ │
│ │ Asset Search          │     │ Admission        │ │
│ └──────────┬────────────┘     └─────────┬────────┘ │
│            │                            │          │
│            └──────────────┬─────────────┘          │
│                           ▼                        │
│ [3] WorldRuntime                                   │
│ ┌────────────────────────────────────────────────┐ │
│ │ Instance Store / SceneGraph                    │ │
│ │ Physics / Spatial / Navigation / Locomotion    │ │
│ │ Interaction                                    │ │
│ └───────────────────────┬────────────────────────┘ │
│                         ▼                          │
│ [4] Verification                                  │
│ ┌────────────────────────────────────────────────┐ │
│ │ desired ↔ observed                            │ │
│ │ relation / interaction / acceptance findings │ │
│ └────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

**Owns**

- Artifact → Asset 的语义编译与准入。
- reusable Asset identity / metadata / physics / actions / provenance。
- Asset Repository / Search。
- World IR / World Compiler / WorldRuntime。
- Physics / Navigation / Interaction / Runtime verification。

**Does Not Own**

- Agent planning / LLM / VLM / Skill workflow。
- modal-2D / modal-3D / Kaggle 等生成 Provider 的选择和 Job lifecycle。
- Modal credential、GPU model cache、Provider 私有数据库。
- Human Project / candidate selection。

**核心边界**

```text
Provider Artifact
      │
      ▼
AgentScape Artifact Admission
      │
      ▼
Asset Compiler
      │
      ▼
Asset Repository
      │
      ▼
World references Asset
```

**已验证模块边界（2026-08-28）**

```text
Asset Module
  owns ArtifactRegistry / ArtifactByteStore
  owns AssetManager / CompiledAssetStore / AssetCatalog
  exposes publishAsset({ artifactId, assetId, label })
  creates state only through createAssetModule()
        │
        │ AssetRef { assetId }
        ▼
World Core
  consumes injected Asset module
  WorldRuntime imports no Asset implementation
  World execution entities carry AssetRef only

Authoring Compatibility Shell (`src/authoring/`)
  owns ProviderRegistry / Connector / GenerationOrchestrator / GenerationJobCenter
  owns legacy AssetGenerationPort / generated-manifest admission path

AssetCatalog
  single Asset read API
  owns list/search/get/resolveExisting surface
```

已完成：

```text
WorldRuntime → ConnectorClient            REMOVED
WorldRuntime → GenerationOrchestrator     REMOVED
WorldRuntime → Provider/Compiler authoring REMOVED
AssetLibrary                             REMOVED
AssetCatalog                              SINGLE READ API
GenerationOrchestrator → AssetCompiler     REMOVED
World execution → query/generate/provider REMOVED
```

仍待迁移：

```text
GenerationOrchestrator retirement
legacy WorldIR authoring request fields retirement after caller parity
```

**架构参考**：DDD Modular Monolith、OpenUSD Asset/Scene composition、MLIR compiler passes、Kubernetes desired/observed reconciliation。

**Verdict：PURIFY。Agent 逻辑移出；Core 专注 Asset + World。**

---

# CARD 02 — AgentScape-agent

**定位：Agentic Orchestration Caller。**

这是未来 Text → 3D → Asset → World 的主要自动化入口。

```text
                         User Text
                            │
                            ▼
┌────────────────── AgentScape-agent ──────────────────┐
│                                                     │
│ [1] Agent Interface                                 │
│ ┌─────────────────────────────────────────────────┐ │
│ │ task / conversation / optional references      │ │
│ └──────────────────────┬──────────────────────────┘ │
│                        ▼                            │
│ [2] Agent Runtime                                   │
│ ┌─────────────────────────────────────────────────┐ │
│ │ LLM/VLM                                         │ │
│ │ Agent Run                                       │ │
│ │ planning / checkpoint / retry / tool trace     │ │
│ └──────────────────────┬──────────────────────────┘ │
│                        ▼                            │
│ [3] Skills                                           │
│ ┌─────────────────┬─────────────────┬─────────────┐ │
│ │ source_3d_asset │ build_world     │ interact    │ │
│ └────────┬────────┴────────┬────────┴──────┬──────┘ │
│          │                 │               │        │
│          ▼                 ▼               ▼        │
│ [4] Deterministic Tool Workflows                     │
│                                                     │
│ source_3d_asset:                                    │
│ search asset                                       │
│     │ miss                                         │
│     ▼                                              │
│ generate N images ───────► modal-2D-client         │
│     │                                              │
│     ▼                                              │
│ VLM evaluate / rank / retry                        │
│     │                                              │
│     ▼                                              │
│ selected image                                     │
│     │                                              │
│     ▼                                              │
│ generate 3D ─────────────► modal-3D-client         │
│     │                                              │
│     ▼                                              │
│ verify Artifact                                    │
│     │                                              │
│     ▼                                              │
│ publish_asset ────────────► AgentScape             │
│                                                     │
│ build_world:                                        │
│ semantic relations ───────► AgentScape World API   │
└─────────────────────────────────────────────────────┘
```

**Owns**

- Agent Run / checkpoint / task progress。
- LLM/VLM decision trace。
- Skill orchestration。
- Candidate generation strategy、ranking、retry、selection。
- 对外 Job/Artifact/Asset/World ID 的引用关系。

**Does Not Own**

- Provider canonical execution truth。
- PNG/GLB content store。
- Provider rembg/model runtime。
- Asset semantic repository。
- Live World state / physics。

**Text→3D 定案**

默认不是一个低级 Tool：

```text
source_3d_asset(text)
  ├─ search_assets
  ├─ generate_images(N)
  ├─ evaluate_candidates
  ├─ generate_3d(selected)
  ├─ verify
  └─ publish_asset
```

只有未来真正存在原生 Text→3D Provider 时，才增加 `generate_3d_from_text` Tool；Skill 可自主选择 native 或 image-mediated path。

**Agent vs Tool**

```text
Agent decides:
目标、候选是否够好、是否重试、采用哪个结果

Deterministic workflow handles:
submit / wait / poll / download / hash / contract validation
```

避免 LLM 每一步都 poll/download。

**推荐初始代码形态**

```text
agentscape_agent/
  agent.py          # Agent run + tool loop
  skills.py         # 少量高层 skill
  tools.py          # stable tool surface
  runs.py           # durable run/checkpoint
  adapters.py       # AgentScape / 2D / 3D / VLM ports
```

先保持单文件高内聚；只有 Skill workflow/Run state 真正增长后再拆。

**架构参考**：LangGraph 的 durable agent/workflow 思维；不要求引入 LangGraph 依赖。

**Verdict：NEW REPOSITORY。Agent 从 AgentScape Core 正式分离。**

---

# PACKAGE CARD — AgentScape/sdk/python

**定位：AgentScape monorepo 内的第一方 Domain Reference SDK / CLI package。**

```text
                    Developer / CI / Agent Adapter
                              │
                              ▼
┌──────────── AgentScape/sdk/python package ───────────┐
│                                                     │
│ [1] Public SDK / CLI                                │
│                                                     │
│ [2] Domain Contracts                                │
│ ┌──────────────────┬──────────────────────────────┐ │
│ │ Asset            │ World / Runtime             │ │
│ │ search/get       │ create/compile/observe      │ │
│ │ publish/admit    │ place/interact/verify       │ │
│ └────────┬─────────┴──────────────┬───────────────┘ │
│          │                        │                 │
│          ▼                        ▼                 │
│ [3] Transport Adapter                               │
│ direct / HTTP                                       │
│          │                                          │
│          ▼                                          │
│ [4] Contract / Artifact helper                      │
│ serialization / validation                          │
└──────────┬──────────────────────────────────────────┘
           ▼
       AgentScape
```

**Owns**：AgentScape 公开 domain contract 的调用体验、CLI、reference examples、contract validation。源码 ownership 属于 `AgentScape` monorepo，不再是独立 repository。

**Does Not Own**

- modal-2D/modal-3D Provider 调用。
- Agent loop。
- Asset/World canonical state。
- global generation orchestration。

**重要修正**：该 package 不再做“所有 Provider 的万能 Client”。Provider 已经有 `modal-2D-client` / `modal-3D-client` 等独立 Reference Sidecar；distribution 名 `agentscape-client` 仅作为兼容 package identity 保留。

**状态**：无业务持久状态；允许 CLI 临时 cache/session。

**目标代码形态**：`client.py / assets.py / worlds.py / runtime.py / cli.py`，不要提前制造 Manager/Repository/Factory。

**Verdict：MERGED INTO AGENTSCAPE MONOREPO。`sdk/python` 是唯一源码真相源；独立 `AgentScape-client` repository 已删除。**

---

# CARD 04 — modal-inference-hub

**定位：Human Workflow Caller / Local Inference Workspace。**

它不是 modal-2D/modal-3D 的唯一上级，而是与 `AgentScape-agent` 平级的人类调用入口。

```text
                      Human User
                          │
                          ▼
┌──────────────── modal-inference-hub ────────────────┐
│                                                    │
│ [1] UI / Workspace                                 │
│ ┌────────────────────────────────────────────────┐ │
│ │ Projects / previews / history / inspection   │ │
│ └──────────────────────┬─────────────────────────┘ │
│                        ▼                           │
│ [2] Project Domain                                  │
│ ┌────────────────────────────────────────────────┐ │
│ │ source images                                  │ │
│ │ generated candidates                          │ │
│ │ selected candidate                            │ │
│ │ masks / manual edits                          │ │
│ │ workflow history                              │ │
│ └───────────────┬───────────────────┬────────────┘ │
│                 │                   │              │
│                 ▼                   ▼              │
│ [3A] Human Semantic Prep     [3B] Workflow Compose │
│ ┌──────────────────────┐    ┌────────────────────┐ │
│ │ object selection     │    │ text→N images     │ │
│ │ component selection  │    │ image→3D          │ │
│ │ manual mask/edit     │    │ retry/history     │ │
│ └──────────┬───────────┘    └─────────┬──────────┘ │
│            │                          │            │
│            └──────────────┬───────────┘            │
│                           ▼                        │
│ [4] Sidecar Adapters                               │
│       ┌────────────────┬────────────────┐          │
│       ▼                ▼                ▼          │
│ modal-2D-client  modal-3D-client   AgentScape?    │
│       │                │          publish optional │
└───────┼────────────────┼───────────────────────────┘
        ▼                ▼
     modal-2D         modal-3D
```

**Owns**

- Project/workspace state。
- Human candidate selection。
- Human semantic preprocess：哪个物体、哪个 component、手工 mask。
- 生成历史与本地实验 UX。
- 人类工作流 composition。

**Does Not Own**

- modal Provider 私有 Job/Volume/model lifecycle。
- model-required automatic rembg/input canonicalization 的最终正确性。
- Agent planning。
- AgentScape Asset/World canonical truth。

**Human Text→3D**

```text
prompt
  ↓
modal-2D-client → N candidates
  ↓
human select/edit
  ↓
modal-3D-client
  ↓
GLB
  ↓
optional publish → AgentScape Asset Repository
```

**当前迁移**

旧仓内部仍直接拥有大量 3D Job/Artifact/Modal session 代码；目标是逐步改为调用新 `modal-3D-client`，随后 2D composition 改为调用 `modal-2D-client`。Project/Preprocess/UI 保留。

**Verdict：RENAMED + REPOSITION。原 modal-3D-client 演化为 Human Caller Hub。**

---

# CARD 05 — modal-gen-client

**定位：Optional Local Security Gateway。**

它不是生成业务层，也不是 Agent/Hub 的共同上级；只有 Browser/WebView 需要本机特权隔离时才出现。

```text
                    Browser / WebView
                         │ untrusted
                         ▼
┌──────────────── modal-gen-client ────────────────┐
│ [1] Security Boundary                           │
│ pairing / origin / scope / session              │
│                    │                            │
│                    ▼                            │
│ [2] Credential Boundary                         │
│ privileged secret never returned to browser    │
│                    │                            │
│                    ▼                            │
│ [3] Mechanical Transport Router                 │
│ operation → local sidecar endpoint              │
│          ┌───────────┴────────────┐             │
│          ▼                        ▼             │
│ modal-2D-client             modal-3D-client     │
│                                                  │
│ [4] Local Projection                             │
│ session / transport mapping only                │
└──────────────────────────────────────────────────┘
```

**Owns**：pairing、origin、scope、session、secret isolation、loopback transport authorization。

**Does Not Own**：candidate selection、Text→3D composition、Project、Provider job truth、Asset/World、Agent。

**Rule**

```text
Browser deployment:
UI → modal-gen-client → sidecar

Server/Agent deployment:
Caller → sidecar directly
```

因此它永远是可选 Adapter，不是主系统中枢。

**Verdict：KEEP + HARD SHRINK / OPTIONAL TRANSPORT。**

---

# CARD 06 — modal-2D-client

**定位：Image Provider Reference Sidecar。**

```text
                 Caller / Security Gateway
                          │
                          ▼
┌────────────────── modal-2D-client ──────────────────┐
│ [1] Local API                                      │
│ submit / status / cancel / artifact                │
│                       │                             │
│                       ▼                             │
│ [2] Durable Execution Mirror                       │
│ request identity / remote binding / restart       │
│                       │                             │
│                       ▼                             │
│ [3] modal-2D Adapter                               │
│ capability / submit / poll                         │
│                       │                             │
│                       ▼                             │
│ [4] Artifact Fetch                                 │
│ named Volume first                                 │
│      │                                             │
│      ├─ stream                                     │
│      ├─ PNG/bytes/SHA verify                       │
│      └─ content-addressed local cache              │
│                       │                             │
│ legacy read_artifact Function = fallback only     │
└───────────────────────┬─────────────────────────────┘
                        ▼
                     modal-2D
```

**Owns**：本地 Job mirror、request identity、remote binding/recovery、Provider-specific Artifact transport、verified local cache。

**Does Not Own**：prompt semantic strategy、candidate ranking、Asset、World、模型/GPU。

**已验证迁移事实（040 前置基线）**

- Volume-first Artifact fetch 已真实通过。
- legacy Function fallback 保留兼容。
- content integrity fail-closed。

**目标稳定 API**

```text
prompt + model/profile/options
        ↓
local execution projection
        ↓
verified image Artifact
```

**Verdict：KEEP / STABLE REFERENCE SIDECAR。**

---

# CARD 07 — modal-2D

**定位：Pure Image Generation Provider。**

```text
                      Prompt Request
                           │
                           ▼
┌──────────────────── modal-2D ─────────────────────┐
│ [1] Provider Contract                            │
│ image.generate                                   │
│ input schema / models / readiness                │
│                      │                           │
│                      ▼                           │
│ [2] Model Runtime                                │
│ model definition / weights / preload / cache    │
│                      │                           │
│          ┌───────────┴───────────┐               │
│          ▼                       ▼               │
│ SANA-Sprint 0.6B          SANA-Sprint 1.6B      │
│          │                       │               │
│          └───────────┬───────────┘               │
│                      ▼                           │
│ [3] PNG Artifact                                 │
│ header/IHDR/dimensions                           │
│ mediaType / bytes / digest / producer            │
│                      │                           │
│                      ▼                           │
│ [4] Provider-private Artifact Volume             │
└───────────────────────────────────────────────────┘
```

**Owns**：Image model/runtime、GPU inference、weights/cache/readiness、PNG generation、Producer Artifact identity。

**Does Not Own**：candidate selection、Text→3D workflow、Project、Asset/World。

**040 实验 Gate**

```text
2 models × 2 seeds
real GPU
real PNG
Volume read
1024×1024
bytes/digest/producer verify
```

实验通过后两个模型才可标记为 `verified`，而不是因为 capability 声明存在就认为可用。

**架构原则**：Provider 越窄越好，不添加 Project/Agent/Workflow DB。

**Verdict：KEEP SIMPLE。**

---

# CARD 08 — modal-3D-client

**定位：3D Provider Reference Sidecar。**

新的独立仓库与 `modal-2D-client` 对称。它不是原来的 Project/Preprocess/UI 应用。

```text
                 Caller / Security Gateway
                          │
               image + optional mask
                          ▼
┌────────────────── modal-3D-client ──────────────────┐
│ [1] Local API                                      │
│ submit / status / cancel / artifact / models      │
│                       │                             │
│                       ▼                             │
│ [2] Input Transport Validation                     │
│ supported media type / size limit                 │
│ preserve original image + optional mask           │
│                       │                             │
│                       ▼                             │
│ [3] Durable Execution Mirror                       │
│ request identity                                   │
│ remote task binding                                │
│ uncertain submit / idempotent rebind              │
│ restart recovery                                   │
│                       │                             │
│                       ▼                             │
│ [4] modal-3D Adapter                               │
│ capability / model / submit / cancel              │
│                       │                             │
│                       ▼                             │
│ [5] GLB Artifact Fetch                             │
│ Provider Volume                                    │
│    ↓                                               │
│ glTF Binary v2 / bytes / SHA verify               │
│    ↓                                               │
│ content-addressed cache                            │
└───────────────────────┬─────────────────────────────┘
                        ▼
                     modal-3D
```

**Owns**：Provider-specific transport、local durable execution projection、upload/download、GLB validation/cache。

**Does Not Own**

- Project / UI / candidate selection。
- semantic component selection。
- rembg 模型或 model-required crop/normalize logic。
- Agent planning / Asset semantics。

**重要目标修正**

当前新仓初版仍只接受 `1024×1024 RGBA canonical PNG`。这是过窄的过渡 contract，目标要改成：

```text
supported image bytes
+ mediaType
+ optional mask/alpha
        ↓
upload unchanged
        ↓
modal-3D InputConditioner
```

Sidecar 可以做安全/格式验证，但不能要求所有 Caller 理解 TRELLIS/Hunyuan/SAM3D 的内部 canonical quirks。

**Durable Submit**

```text
stable request id
   ↓
content-addressed input upload
   ↓
gateway submit
   │ network outcome unknown
   ▼
retry SAME model + input + options
   ↓
provider idempotent rebind
```

**Verdict：NEW REPOSITORY / PURE REFERENCE SIDECAR。**

---

# CARD 09 — modal-3D

**定位：Multi-model 3D Provider + Model Input Conditioning。**

```text
                 Image + optional alpha/mask
                           │
                           ▼
┌──────────────────── modal-3D ──────────────────────┐
│ [1] Provider API                                  │
│ capability / models / submit                      │
│                       │                           │
│                       ▼                           │
│ [2] InputConditioner                              │
│ ┌───────────────────────────────────────────────┐ │
│ │ decode / orientation / color                 │ │
│ │                  │                           │ │
│ │         valid alpha/mask?                    │ │
│ │        ┌─────────┴─────────┐                 │ │
│ │        ▼                   ▼                 │ │
│ │     preserve       segmentation/rembg        │ │
│ │        └─────────┬─────────┘                 │ │
│ │                  ▼                           │ │
│ │       subject bbox / crop / center           │ │
│ │                  ▼                           │ │
│ │        model-private canonical input         │ │
│ └──────────────────┬────────────────────────────┘ │
│                    ▼                              │
│ [3] Model Registry / Profiles                     │
│                    │                              │
│       ┌────────────┼─────────────┬────────────┐   │
│       ▼            ▼             ▼            ▼   │
│ FastSAM3D++  TRELLIS2++   Hunyuan2.1++    Pixal3D│
│ Worker       Worker        Worker           Worker│
│       └────────────┼─────────────┴────────────┘   │
│                    ▼                              │
│ [4] GLB Artifact                                  │
│ id / role / mediaType / bytes / digest / producer│
│                    │                              │
│                    ▼                              │
│ [5] Provider-private Artifact Volume              │
└────────────────────────────────────────────────────┘
```

**Gateway Owns**：public input contract、model/profile registry、readiness、dispatch、provider-level idempotency。

**InputConditioner Owns**：满足 Provider/模型输入 invariant 的自动处理。`rembg` 只是策略之一。

**Worker Owns**：模型加载、weights、GPU lifecycle、model-specific preprocess/postprocess、模型私有 canonical representation。

**Does Not Own**：Human/Agent 语义选择、Project、Asset/World。

**Public vs Internal Contract**

目标：

```text
PUBLIC:
image/* + optional mask/alpha

INTERNAL worker contract:
model-required canonical image/mask
```

当前部署仍公开要求 `1024×1024 RGBA alpha_required`；`041` 实验将它记录为迁移前 baseline。随后 InputConditioner 下沉到 Provider 后再放宽 public contract。

**041 实验 Gate**

- real BiRefNet/current preprocessing evidence。
- 同一 canonical 输入进入 4 个 enabled models。
- gateway duplicate submit 返回同一 call ID。
- 每个模型产生真实 GLB。
- GLB magic/version/declared bytes/SHA 全匹配。

**Verdict：KEEP MULTI-MODEL STRUCTURE；ADD INPUT CONDITIONER，DON'T CENTRALIZE WORKERS。**

---

# CARD 10 — kaggle-inference-hub

**定位：Queue-backed Execution Provider。**

```text
                      Caller
                        │
                        ▼
┌──────────── kaggle-inference-hub ────────────────┐
│ [1] Consumer API                                │
│ submit / status / cancel? / artifact            │
│                      │                           │
│                      ▼                           │
│ [2] Task Core                                   │
│ state machine / queue / retry / terminal state │
│                      │                           │
│                      ▼                           │
│ [3] Lease / Worker Coordination                 │
│ register / claim / lease / heartbeat / reclaim │
│              ┌───────┴────────┐                 │
│              ▼                ▼                 │
│           Worker A         Worker B             │
│              └───────┬────────┘                 │
│                      ▼                           │
│ [4] Worker Result Protocol                      │
│ upload / complete / fail                        │
│                      │                           │
│                      ▼                           │
│ [5] Artifact Store                              │
│ immutable result / digest / metadata            │
│                                                 │
│ [外围] Prompt UI / WebSocket / History View     │
│       只消费 Consumer API                       │
└─────────────────────────────────────────────────┘
```

**Owns**：task queue、lease、worker heartbeat/reclaim、retry、completion、artifact binding。

**关键边界**：Consumer Protocol ≠ Worker Protocol。Caller 不看 lease/heartbeat；Worker 不看 Agent/Asset/World 语义。

**Caller 关系**：`AgentScape-agent`、`modal-inference-hub` 或测试程序都可以通过独立 Adapter 调它；没有唯一上级。

**目标代码形态**：`app.py / state.py / consumer_api.py / worker_api.py / artifacts.py / prompt_pipeline.py`，到这里停止。

**参考**：Celery producer/queue/worker、lease/recovery patterns。

**Verdict：MAJOR INTERNAL REWRITE / QUEUE CORE。**

---

# CARD 11 — modal-build

**定位：Reproducible Build + Embodied Runtime Distribution。**

```text
                    Upstream Sources
                  EmbodiedGen / others
                          │
                          ▼
┌──────────────────── modal-build ─────────────────────┐
│                  BUILD PLANE                        │
│ [1] Source Definition                              │
│ upstream URL / pinned revision                     │
│                    │                               │
│                    ▼                               │
│ [2] Patch Inventory                                │
│ explicit patch / patch digest                      │
│                    │                               │
│                    ▼                               │
│ [3] Build Definition                               │
│ Python/CUDA/system deps / wheels / weights        │
│                    │                               │
│                    ▼                               │
│ [4] Build Verification                             │
│ import/ABI/runtime smoke / output digest           │
│                    │                               │
│                    ▼                               │
│          Immutable Runtime Artifact                │
├────────────────────┼───────────────────────────────┤
│                 RUNTIME PLANE                       │
│                    ▼                               │
│ [5] Provider API / Execution                       │
│       ┌──────────┬──────────┬──────────┐           │
│       ▼          ▼          ▼          ▼           │
│    Image      Image→3D   Retexture  Affordance    │
│    Worker      Worker      Worker      Worker      │
│       └──────────┴──────┬───┴──────────┘           │
│                         ▼                           │
│                    Artifacts/Evidence              │
└─────────────────────────────────────────────────────┘
```

**Build Plane Owns**：source revision、patches、dependencies、weights、runtime image identity、build verification。

**Runtime Plane Owns**：GPU execution、runtime job/artifact、Provider evidence。

**Does Not Own**：Agent decision、Asset/World semantic truth。

**规则**：Runtime request 不允许临时 build/patch 本应离线固定的 CUDA/ABI dependency。

大型 runtime 按真实 GPU lifecycle 拆，而不是按 utils/helper 拆。

**参考**：Nix derivation、Bazel action/artifact、OCI digest identity。

**Verdict：MAJOR REWRITE / BUILD PLANE + RUNTIME PLANE。**

---

# CARD 12 — EmbodiedGen

**定位：Pinned Readonly Upstream。**

```text
┌─────────────────────────────┐
│ HorizonRobotics/EmbodiedGen │
│ upstream algorithms/package │
└──────────────┬──────────────┘
               │ pinned revision
               ▼
┌─────────────────────────────┐
│         modal-build         │
│ patch / build / wrap / test │
└──────────────┬──────────────┘
               ▼
         Provider Runtime
```

**规则**：PIN → PATCH OUTSIDE → WRAP → VERIFY。

**禁止**

- AgentScape / AgentScape-agent 直接 import upstream internals。
- 为了下游 Contract 重构 upstream。
- 把 upstream source 当 production runtime state。

**Compatibility Owner**：全部归 `modal-build`。

**Verdict：DO NOT REWRITE / READONLY UPSTREAM。**

---

# CARD 13 — modal-lab

**定位：Research + Verification Incubator。**

`modal-lab` 不只是模型玩具区，也负责在 Architecture Migration 前验证“某项 capability 真的可用”。

```text
                     Research Question
                           │
                           ▼
┌──────────────────── modal-lab ─────────────────────┐
│ [1] Experiment Definition                        │
│ upstream / hypothesis / input matrix / gate      │
│                    │                              │
│                    ▼                              │
│ [2] Real Run                                     │
│ GPU / Provider / params / environment            │
│                    │                              │
│                    ▼                              │
│ [3] Evidence                                     │
│ metrics / Artifact / digest / logs / cost        │
│                    │                              │
│                    ▼                              │
│ [4] Verification                                 │
│ contract / content / behavior                    │
│                    │                              │
│           ┌────────┴─────────┐                    │
│           ▼                  ▼                    │
│         FAIL               PASS                   │
│      evidence stays          │                    │
│                              ▼                    │
│ [5] Promotion Gate                               │
└──────────────────────────────┬─────────────────────┘
                               ▼
                    Production Architecture
```

**Owns**

- experiments / benchmark / real-provider verification。
- 模型可行性与 cost/performance evidence。
- migration baseline experiments。

**Does Not Own**：production contract/API、stable runtime dependency、Asset/World state。

**新实验基线**

```text
040-modal-2d-provider
  → 2 models × seeds / PNG / Artifact integrity

041-modal-3d-provider
  → rembg/canonical + 4 model matrix / idempotency / GLB integrity
```

这些实验结果可以决定 Provider 是否标记 `verified`，但 production runtime 不 import 实验代码。

**Promotion**

```text
Experiment
  ↓ reproducible + PASS
Architecture decision
  ↓
production implementation
  ↓
independent smoke
```

**参考**：MLflow Experiment/Run/Artifact 思维；保持当前“一实验一目录”的高内聚结构。

**Verdict：KEEP LOOSE + ADD VERIFICATION ROLE。**

---

# CARD 14 — AgentScape-plan

**定位：Architecture Authority + Migration Ledger。**

```text
                  Architecture Change
                         │
                         ▼
┌────────────────── AgentScape-plan ──────────────────┐
│ [1] System Landscape                              │
│ caller/capability/asset/world boundaries          │
│                    │                              │
│                    ▼                              │
│ [2] Repository Cards                              │
│ owns / does-not-own / state / failure / API      │
│                    │                              │
│                    ▼                              │
│ [3] Shared Contracts + Runtime Views              │
│ request / artifact / asset / world / retry       │
│                    │                              │
│                    ▼                              │
│ [4] Integration Ledger                           │
│ real arrow / target arrow / verdict              │
│                    │                              │
│                    ▼                              │
│ [5] ADR + Migration                               │
│ current → target / gates / evidence               │
│                    │                              │
│                    ▼                              │
│              Code + modal-lab experiments        │
│                    │                              │
│                    ▼                              │
│               Reality Validation                 │
└───────────────────────────────────────────────────┘
```

**Owns**：系统边界、Repository Cards、Shared Contracts、Runtime Views、Integration Ledger、Migration Roadmap、ADR、Verified Baseline。

**Does Not Own**：零散 debug TODO、产品脑暴、未经实验验证却冒充事实的 capability 状态。

**架构变化规则**

当实验/真实代码推翻 Card 假设时：

```text
Evidence
  ↓
update Card / ADR
  ↓
update migration target
  ↓
code change
```

不是让代码强行迁就旧文档。

**Verdict：KEEP AS SINGLE ARCHITECTURE AUTHORITY。**

---

# 汇总判定

| Repository | Target | 重写程度 |
|---|---|---:|
| `AgentScape` | Asset + World Domain Core + `sdk/python` first-party SDK/CLI package | 高：移出 Agent/Generation；吸收重复 SDK repo |
| `AgentScape-agent` | Agentic Orchestration Caller | **新建** |
| `modal-inference-hub` | Human Workflow Caller | 高：从旧 3D Client 提纯 |
| `modal-gen-client` | Optional Local Security Gateway | 高度收缩 |
| `modal-2D-client` | Image Provider Reference Sidecar | 已基本稳定 |
| `modal-2D` | Simple Image Provider | 已基本稳定 |
| `modal-3D-client` | 3D Provider Reference Sidecar | 新仓；需放宽 input contract |
| `modal-3D` | Multi-model Provider + InputConditioner | 中 |
| `kaggle-inference-hub` | Consumer API + Queue/Lease + Worker API | 高 |
| `modal-build` | Build Plane + Runtime Plane | 高 |
| `EmbodiedGen` | Pinned Upstream | 不重写 |
| `modal-lab` | Research + Verification Incubator | 低 |
| `AgentScape-plan` | Architecture Authority | 持续维护 |
