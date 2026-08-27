# Repository Architecture Cards

本文件是仓库级目标架构权威。图中的模块表示职责边界，**不等于必须一个模块一个文件**；能在一个文件内高内聚实现时优先保持简单。

# CARD 01 — AgentScape

**定位：领域核心 / 聚合器 / 模块化单体。**

```text
┌────────────────────── AgentScape ──────────────────────┐
│ [1] Agent                                             │
│   planning / tool calling / decision trace            │
│                    │                                  │
│                    ▼                                  │
│ [2] Skills / Application                              │
│          ┌─────────┴─────────┐                        │
│          ▼                   ▼                        │
│ [3A] Asset Sourcing     [3B] World Intent             │
│ existing / external     World IR / relations          │
│          │                   │                        │
│          ▼                   ▼                        │
│ [4] Artifact/Asset      [5] World Compiler            │
│ verify / compiler       layout/behavior/physics       │
│          └─────────┬─────────┘                        │
│                    ▼                                  │
│ [6] WorldRuntime                                      │
│ store / scene / physics / spatial / nav / interaction│
│                    │                                  │
│                    ▼                                  │
│ [7] Verification                                     │
│ desired ↔ observed / findings / acceptance           │
└────────────────────┬──────────────────────────────────┘
                     │ domain Port
                     ▼
               External Adapter
```

**Owns**：Agent、Skills、Asset semantic truth、Asset Compiler、World IR/Compiler、WorldRuntime、Physics/Nav/Interaction、Runtime verification。

**Does Not Own**：Modal/Kaggle/HF credential、Provider 私有 Job/DB/model cache/CUDA lifecycle。

**目标边界**：

```text
当前：WorldRuntime → ConnectorClient → GenerationOrchestrator
目标：AssetSourcing → Port → Adapter → Provider
```

WorldRuntime 必须彻底不知道 Provider。

**参考**：Terraform Core/Provider、MLIR Pass Pipeline、Kubernetes Controller。

**Verdict：KEEP + PURIFY / Modular Monolith。**

---

# CARD 02 — AgentScape-client

**定位：Reference SDK + CLI + Contract/Artifact Harness。**

```text
                Developer / CI
                     │
                     ▼
┌──────────── AgentScape-client ────────────┐
│ [1] Public API / CLI                     │
│             │                            │
│             ▼                            │
│ [2] Contract Layer                       │
│ request / capability / execution/artifact│
│             │                            │
│             ▼                            │
│ [3] Transport Adapters                   │
│ direct / HTTP / local gateway            │
│             │                            │
│             ▼                            │
│ [4] Artifact Verification                │
│ MIME / bytes / digest / PNG / GLB        │
└─────────────┬────────────────────────────┘
              ▼
           Provider
```

**Owns**：公开 Contract、request 构造、reference invocation、artifact verifier、CLI/E2E harness。

**Does Not Own**：Provider execution truth、Asset/World truth、global routing policy、业务数据库。

**状态**：原则上无业务持久状态；允许临时 poll/session/cache。

**目标代码形态**：少量直接模块，例如 `client.py / contracts.py / artifacts.py / cli.py`；不提前引入 Manager/Repository/Factory。

**参考**：Triton client/server separation。

**Verdict：KEEP + REPOSITION AS REFERENCE CLIENT。**

---

# CARD 03 — modal-gen-client

**定位：Local Security Gateway。**

```text
                  Browser / WebView
                       │ 不可信前端
                       ▼
┌────────────── modal-gen-client ───────────────┐
│ [1] Security Boundary                        │
│ pairing / origin binding / scope / session   │
│                    │                         │
│                    ▼                         │
│ [2] Credential Boundary                      │
│ privileged secret 永远不返回 browser         │
│                    │                         │
│                    ▼                         │
│ [3] Mechanical Transport Router              │
│ operation → configured adapter               │
│          ┌───────────┴───────────┐           │
│          ▼                       ▼           │
│ [4A] 2D Adapter             [4B] 3D Adapter  │
│          │                       │           │
│          ▼                       ▼           │
│ modal-2D-client          modal-3D-client     │
│                                              │
│ [5] Local Projection                         │
│ session / transport job / artifact mapping  │
│ 仅投影，不是业务真值                         │
└──────────────────────────────────────────────┘
```

**Owns**：pairing、origin、scope、session、secret isolation、transport endpoint binding。

**Does Not Own**：generate_asset workflow、Provider 业务选择、Asset/World/Agent。

**Job 规则**：Gateway Job 只能是 Provider execution 的 local projection/cache。

**目标**：路由尽可能机械；业务 composition 留在 caller/AgentScape。

**参考**：Tauri capability security / privileged native boundary。

**Verdict：KEEP + HARD SHRINK。**

---

# CARD 04 — modal-2D-client

**定位：modal-2D Reference Sidecar。**

```text
                 Caller / Gateway
                       │
                       ▼
┌────────────── modal-2D-client ───────────────┐
│ [1] Local API                               │
│ submit / status / cancel / artifact         │
│                    │                        │
│                    ▼                        │
│ [2] Durable Local Mirror                    │
│ request_id / remote binding / state         │
│ restart recovery                            │
│                    │                        │
│                    ▼                        │
│ [3] Modal Transport                         │
│ submit / poll / cancel remote execution     │
└────────────────────┬────────────────────────┘
                     ▼
                  modal-2D
                     │
                     ▼
               PNG Artifact
```

**Owns**：local session、remote invocation binding、poll/cancel/restart recovery、artifact relay。

**Does Not Own**：模型、global provider routing、Asset/World、产品 UI。

**State Truth**：remote Provider execution 是 canonical；sidecar 保存 durable mirror。

**独立完成条件**：start → submit → restart → recover same execution → retrieve/verify PNG。

**Verdict：KEEP + FREEZE SCOPE。**

---

# CARD 05 — modal-2D

**定位：纯 Image Generation Provider。**

```text
                  prompt + options
                        │
                        ▼
┌────────────────── modal-2D ──────────────────┐
│ [1] API / Contract                           │
│ validate / health / models / generation      │
│                    │                         │
│                    ▼                         │
│ [2] Model Runtime                            │
│ model definition / weights / cache           │
│ readiness / preload                          │
│                    │                         │
│                    ▼                         │
│ [3] GPU Inference                            │
│ prompt / seed / guidance / inference         │
│                    │                         │
│                    ▼                         │
│ [4] Artifact Encoder                         │
│ image → PNG bytes + metadata                 │
└────────────────────┬─────────────────────────┘
                     ▼
               image/png Artifact
```

**Owns**：模型、weights/cache、Modal GPU runtime、generation params、PNG production、readiness。

**Does Not Own**：Project、Asset、Agent、World、multi-provider routing、client session。

**State**：仅模型缓存/GPU/readiness，不引入业务 DB。

**Smoke**：prompt → real GPU → PNG → signature/dimensions/digest PASS。

**参考**：KServe/Triton model serving 的 API → model runtime → backend → output 思路。

**Verdict：KEEP ALMOST AS-IS。**

---

# CARD 06 — modal-3D-client

**定位：Local 3D Workflow Host。** 当前已存在 Project、Preprocess、Generation Intent/Job、Connector 等多个真实 State Owner，必须内部重写。

```text
                   Local User / UI / CLI
                           │
                           ▼
┌──────────────── modal-3D-client ───────────────────┐
│ [1] Application Boundary                          │
│ Project API / Preprocess API / Generation API    │
│        ┌──────────────┼──────────────┐             │
│        ▼              ▼              ▼             │
│ [2A] Project    [2B] Preprocess [2C] Generation   │
│ source image    rembg          intent/idempotency  │
│ metadata        segmentation   remote state        │
│ local files     component pick recovery            │
│ history         canonicalize  artifact retrieval   │
│        └──────────────┼──────────────┘             │
│                       ▼                            │
│ [3] Local Artifact Layer                           │
│ source/matte/selection/canonical/GLB              │
│ descriptor + location + digest                    │
│                       │                            │
│                       ▼                            │
│ [4] Provider Adapter                               │
│ modal session / submit / status / cancel / result│
│                       │                            │
│                       ▼                            │
│                    modal-3D                        │
│                                                    │
│ [5] 外围 Adapter                                   │
│ Connector Adapter ─► Application Boundary          │
│ UI Adapter        ─► Application Boundary          │
└────────────────────────────────────────────────────┘
```

**Project Owns**：source、project metadata、local files、history、selected output。

**Preprocess Owns**：rembg、component selection、canonicalization、preprocess verification。

**Generation Saga Owns**：request intent、idempotency、remote-created/uncertain state、provider job binding、restart recovery、artifact retrieval。

**边界规则**：Connector/UI 只能调用 Application API；禁止直接操作其他 Domain Store。

**仓库不拆**：三个内部域仍属于一个 Local 3D workflow product。

**参考**：Temporal durable execution 语义；只学习幂等、uncertain、recovery，不引入 Temporal 服务。

**Verdict：MAJOR INTERNAL REWRITE / ONE REPOSITORY。**

---

# CARD 07 — modal-3D

**定位：Multi-model 3D Inference Provider。**

```text
                Canonical Image Artifact
                          │
                          ▼
┌──────────────────── modal-3D ─────────────────────┐
│ [1] Provider API                                 │
│ /health /models /generation / request validation│
│                        │                         │
│                        ▼                         │
│ [2] Model Registry / Profiles                    │
│ model id / profile / availability / resources   │
│                        │                         │
│                        ▼                         │
│ [3] Model Dispatcher                             │
│       ┌──────────┬──────────┬──────────┐         │
│       ▼          ▼          ▼          ▼         │
│  FastSAM3D++  TRELLIS   Hunyuan    Future       │
│    Worker      Worker     Worker      Worker     │
│       └──────────┴──────┬───┴──────────┘         │
│                         ▼                         │
│ [4] Artifact Normalization                       │
│ ensure GLB / metadata / digest / size            │
└─────────────────────────┬─────────────────────────┘
                          ▼
                  model/gltf-binary
```

**Gateway Owns**：contract validation、model/profile registry、readiness、dispatch。

**Worker Owns**：model loading、weights、GPU requirements、model-specific preprocess/postprocess。

**Does Not Own**：AgentScape Asset semantics、World、Project、browser session。

**Preprocess 规则**：模型专属准备留在 Worker；多个模型共享才提到 modal-3D 公共层；只有跨仓真实复用才考虑独立 Capability/仓库。

**参考**：NVIDIA Triton 的 server / model repository / per-model backend 思路。

**Verdict：KEEP / GATEWAY + MODEL BACKENDS。**

---

# CARD 08 — kaggle-inference-hub

**定位：Queue-backed Execution Provider。**

```text
                      Consumer
                         │
                         ▼
┌──────────── kaggle-inference-hub ────────────────┐
│ [1] Consumer API                                 │
│ submit / status / cancel? / artifact             │
│                       │                          │
│                       ▼                          │
│ [2] Task Core                                    │
│ state machine / queue / retry / terminal state  │
│                       │                          │
│                       ▼                          │
│ [3] Lease / Worker Coordination                  │
│ register / claim / lease / heartbeat / reclaim  │
│              ┌────────┴────────┐                 │
│              ▼                 ▼                 │
│           Worker A          Worker B             │
│              └────────┬────────┘                 │
│                       ▼                          │
│ [4] Worker Result Protocol                       │
│ upload / complete / fail                         │
│                       │                          │
│                       ▼                          │
│ [5] Artifact Store                               │
│ immutable output / digest / metadata             │
│                                                  │
│ [外围] Prompt UI / WebSocket / History View      │
│       只消费 Consumer API，不进入 Task Core      │
└──────────────────────────────────────────────────┘
```

**Hub Core Owns**：task queue、worker、lease、heartbeat、retry、completion state、artifact binding。

**关键边界**：Consumer Protocol ≠ Worker Protocol。Consumer 不知道 claim/heartbeat/lease；Worker 不知道产品/Agent/Asset 语义。

**目标内部形态**：

```text
hub/
  app.py            # composition root
  state.py          # queue/lease truth
  consumer_api.py
  worker_api.py
  artifacts.py
  prompt_pipeline.py # optional helper
```

到这里停止，不再制造 Service/Repository/Manager 套娃。

**参考**：Celery 的 producer/queue/worker 模型与 lease/recovery 思维。

**Verdict：MAJOR INTERNAL REWRITE / QUEUE CORE。**

---

# CARD 09 — modal-build

**定位：Reproducible Build + Embodied Runtime Distribution。** 明确分 Build Plane 与 Runtime Plane。

```text
                    Upstream Sources
                 EmbodiedGen / others
                          │
                          ▼
┌──────────────────── modal-build ─────────────────────┐
│                  BUILD PLANE                         │
│ [1] Source Definition                               │
│ upstream URL / pinned revision                      │
│                     │                               │
│                     ▼                               │
│ [2] Patch Layer                                     │
│ explicit patches / patch digest                     │
│                     │                               │
│                     ▼                               │
│ [3] Build Definition                                │
│ Python/CUDA/system deps / wheels / weights preload │
│                     │                               │
│                     ▼                               │
│ [4] Build Verification                              │
│ import/ABI/runtime smoke / artifact digest          │
│                     │                               │
│                     ▼                               │
│           Immutable Runtime Artifact                │
├─────────────────────┼───────────────────────────────┤
│                  RUNTIME PLANE                       │
│                     ▼                               │
│ [5] Runtime Entry / Job Control                      │
│       ┌─────────┬──────────┬──────────┐             │
│       ▼         ▼          ▼          ▼             │
│    Image     Image→3D   Retexture  Affordance      │
│    Worker     Worker      Worker      Worker        │
│       └─────────┴──────┬───┴──────────┘             │
│                        ▼                             │
│                     Artifacts                        │
└──────────────────────────────────────────────────────┘
```

**Build Plane Owns**：source revision、patch inventory、dependencies、weights preload、build artifact identity、build verification。

**Runtime Plane Owns**：GPU worker lifecycle、runtime executions、runtime artifacts/provider evidence。

**唯一连接**：Build Plane 产出 immutable/runtime-identified artifact，Runtime Plane 消费。Runtime request 不允许临时重建本应离线固定的 CUDA/ABI 依赖。

**目标 Runtime 形态**：

```text
runtime/embodiedgen/
  app.py
  profiles.py
  jobs.py
  artifacts.py
  image.py
  image3d.py
  retexture.py
  affordance.py
```

按真实 GPU lifecycle 拆，不按 utility 拆。

**参考**：Nix reproducible derivation、Bazel artifact/action、OCI digest identity。

**Verdict：MAJOR REWRITE / BUILD PLANE + RUNTIME PLANE。**

---

# CARD 10 — EmbodiedGen

**定位：Pinned Readonly Upstream。**

```text
┌─────────────────────────────┐
│ HorizonRobotics/EmbodiedGen │
│ 上游算法 / 上游 package     │
└──────────────┬──────────────┘
               │ pinned revision
               ▼
┌─────────────────────────────┐
│         modal-build         │
│ clone / patch / build / wrap│
└──────────────┬──────────────┘
               ▼
      Production Runtime
```

**规则**：PIN → PATCH OUTSIDE → WRAP → VERIFY。

**禁止**：AgentScape Runtime 直接 import upstream internals；为了 AgentScape contract 修改 upstream 架构。

**Compatibility Owner**：全部归 `modal-build`。

**Verdict：DO NOT REWRITE / READONLY UPSTREAM。**

---

# CARD 11 — modal-lab

**定位：Research Incubator。** 当前“一实验一目录 + README/UPSTREAM/run/modal_app/benchmark”方向保留。

```text
                      Researcher
                          │
                          ▼
┌──────────────────── modal-lab ────────────────────┐
│ [1] Upstream                                     │
│ revision / model / source                       │
│                     │                            │
│                     ▼                            │
│ [2] Experiment Definition                        │
│ modal_app / run / config                         │
│                     │                            │
│                     ▼                            │
│ [3] Run                                           │
│ GPU / params / environment                       │
│                     │                            │
│                     ▼                            │
│ [4] Evidence                                      │
│ metrics / benchmark / artifacts / cost           │
│                     │                            │
│                     ▼                            │
│ [5] Promotion Decision                            │
└─────────────────────┬────────────────────────────┘
                      │ 值得生产化
                      ▼
               Production Repository
```

**Owns**：实验、benchmark、模型探索、成本/速度/质量证据。

**Does Not Own**：production contract/API、stable Provider、AgentScape Runtime dependency。

**代码规则**：允许高内聚单文件、临时实现和实验参数；必须保留 upstream、运行方法、环境与结果证据。

**Promotion**：Experiment → reproducible → verified → production boundary → production implementation。

**参考**：MLflow Experiment/Run/Artifact；Hydra 只在配置组合真正复杂时采用。

**Verdict：KEEP LOOSE / NEVER PRODUCTION DEPENDENCY。**

---

# CARD 12 — AgentScape-plan

**定位：Architecture Authority + Migration Ledger。**

```text
                  Architecture Change
                         │
                         ▼
┌────────────────── AgentScape-plan ──────────────────┐
│ [1] System Landscape                               │
│ 仓库是谁 / 边界是什么                              │
│                    │                               │
│                    ▼                               │
│ [2] Repository Cards                               │
│ owns / does-not-own / state / API / failure       │
│                    │                               │
│                    ▼                               │
│ [3] Runtime / Integration Views                     │
│ request / artifact / credential / retry 流向       │
│                    │                               │
│                    ▼                               │
│ [4] ADR                                             │
│ architecture-significant decisions only            │
│                    │                               │
│                    ▼                               │
│ [5] Migration                                      │
│ current → target / compatibility / gates           │
│                    │                               │
│                    ▼                               │
│                 Code / Tests                       │
│                    │                               │
│                    ▼                               │
│              Reality Validation                    │
└────────────────────────────────────────────────────┘
```

**Owns**：System Landscape、Repository Cards、Shared Contracts、Runtime Views、Integration Ledger、Migration、ADR、Verified Baseline。

**Does Not Own**：短期 debug TODO、重复 README、未经验证的产品脑暴作为最高权威。

**文档规则**：Card 集中维护，避免“一张卡一个文件”；ADR 只记录真正架构决策；旧 Studio/Companion/PA 路线由 Git 历史追溯，不再保留平行权威。

**参考**：C4 System Landscape、arc42 架构知识分区、ADR decision log。

**Verdict：REWRITE AS ARCHITECTURE AUTHORITY。**

---

# 汇总判定

| Repository | Target | 重写程度 |
|---|---|---:|
| AgentScape | Modular Domain Core | 中高 |
| AgentScape-client | Reference SDK/CLI | 中 |
| modal-gen-client | Local Security Gateway | 高度收缩 |
| modal-2D-client | Reference Sidecar | 低 |
| modal-2D | Simple Image Provider | 极低 |
| modal-3D-client | Project + Preprocess + Generation Saga | 高 |
| modal-3D | Gateway + Registry + Model Workers | 低中 |
| kaggle-inference-hub | Consumer API + Queue/Lease + Worker API | 高 |
| modal-build | Build Plane + Runtime Plane | 高 |
| EmbodiedGen | Pinned Upstream | 不重写 |
| modal-lab | Research Incubator | 不 productionize |
| AgentScape-plan | Architecture Authority | 已重建 |
