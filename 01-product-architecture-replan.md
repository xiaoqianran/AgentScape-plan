# AgentScape Studio × Cloud Providers × Local Companion 产品架构重规划

> **状态：新的跨项目产品架构权威文档**  
> 本文重新定义 `AgentScape`、`modal-2D`、`modal-3D`、`modal-3D-client`、`modal-2D-client`、`modal-gen-client` 的长期职责与调用拓扑。  
> 如本文与 `00-master-roadmap.md`、`client-unification/01-modal-2d-3d-client-unification-plan.md`、`group-3-agentscape/07-agent-native-world-architecture-replan.md` 中关于 **Local Connector / 单一桌面 Connector / Provider 调用拓扑** 的旧描述冲突，**以本文为准**。  
> `07-agent-native-world-architecture-replan.md` 对 World IR、Compiler、Runtime、Physics、Verification 的核心世界架构仍然有效；本文重写的是 **产品面、Frontend、生成 Provider、Local Companion、Asset Catalog 与跨端连接关系**。

---

## 0. 一句话定义

最终产品不是：

```text
AgentScape
   ↓
modal-gen-client
   ↓
modal-3D-client
   ↓
modal-3D
```

而是：

```text
                     ┌─────────────────────────────┐
                     │      AgentScape Studio      │
                     │   用户控制 AI / World / Asset│
                     └──────────────┬──────────────┘
                                    │
                                    ▼
                     ┌─────────────────────────────┐
                     │      AgentScape Cloud       │
                     │ Agent / Skills / World      │
                     │ Assets / Generation         │
                     └──────────┬──────────┬───────┘
                                │          │
                                ▼          ▼
                         ┌──────────┐  ┌──────────┐
                         │ modal-2D │  │ modal-3D │
                         │ 生图     │  │ 生3D     │
                         └────▲─────┘  └────▲─────┘
                              │             │
                              │ 同一 Provider Contract
                              │             │
                         ┌────┴─────────────┴────┐
                         │   Local Companion      │
                         │ 现 modal-3D-client 演进│
                         │ 本机素材/预处理/预览    │
                         │ 也可直接生图、生3D      │
                         └──────────┬─────────────┘
                                    │ metadata / materialize
                                    ▼
                            AgentScape Asset Catalog
```

核心原则：

> **AgentScape Cloud 与 Local Companion 是 `modal-2D` / `modal-3D` 的两个平级消费者。**  
> `modal-2D` / `modal-3D` 是纯云端生成 Provider；Local Companion 不是生成控制面的必经中间层；AgentScape Frontend 也不直接持有 Provider Secret。

---

# 1. 产品目标

系统最终同时满足四个用户故事。

### 1.1 AI 自动创建世界

用户在 AgentScape Studio 输入：

```text
“做一个现代咖啡厅，优先使用我已有的家具，没有的再生成。”
```

Agent 自动：

```text
理解需求
→ 拆分 World Requirements
→ 搜索 Asset Catalog
→ 优先选择已有资产
→ 判断缺失资产
→ 调用生成 Skills
→ modal-2D / modal-3D 生成
→ Artifact 校验
→ Asset Compiler
→ World IR
→ Layout / Behavior / Physics
→ Runtime
→ Verification / Repair
→ World Ready
```

### 1.2 用户在本机管理自己的素材

用户可以在本机 Companion：

- 拖入 PNG/JPG/GLB 等素材；
- 本地做抠图、component selection、canonicalization；
- 查看缩略图与 3D 预览；
- 建立本地项目与收藏；
- 选择哪些素材发布 metadata 给 AgentScape；
- 只在真正需要时上传 bytes。

### 1.3 用户本机也能独立生图、生 3D

Local Companion 不依赖 AgentScape 才能工作：

```text
Local Companion → modal-2D → image
Local Companion → modal-3D → GLB
```

它与 AgentScape Cloud 使用同一套 Provider Contract。

### 1.4 Agent 可以挑选用户本机资产

AgentScape 不读取用户任意文件系统路径。

它只搜索经过 Companion 发布的 Asset metadata：

```text
AgentScape Asset Catalog
  ├── cloud assets
  ├── generated assets
  └── local companion assets（metadata）
```

当 Agent 真正决定使用一个 local-only 资产时，再执行：

```text
materialize(asset)
→ Companion 上传/提供 bytes
→ AgentScape 验证
→ Compiler
→ World
```

---

# 2. 关键架构决策 / ADR

## ADR-P01：Provider 与 Consumer 严格分离

`modal-2D`、`modal-3D` 只提供生成能力，不拥有用户产品状态。

Provider 不知道 AgentScape Scene、World IR、用户本机目录、Local Project、React UI、Agent conversation 或 Asset Catalog 的业务语义。

Provider 只知道：

```text
Capability / Request / Job / Artifact / Error
```

## ADR-P02：AgentScape Cloud 与 Local Companion 平级消费 Provider

```text
错误长期拓扑：
AgentScape → Local Companion → modal-3D

正确长期拓扑：
AgentScape ───────► modal-3D
Local Companion ─► modal-3D
```

## ADR-P03：AgentScape Frontend 只调用 AgentScape Backend

```text
Browser
  ↓
AgentScape API / Realtime
  ↓
Agent Skills / Generation Service
  ↓
modal-2D / modal-3D
```

Browser 不直接持有 Provider Secret，也不承担 Job ownership、policy、cost、retry、audit 和 lineage。

## ADR-P04：`modal-3D-client` 演进为 Local Companion

长期职责：

```text
Local Asset Workstation
+ Local Project Manager
+ Local Preprocessing
+ Artifact Cache
+ Provider Consumer
+ AgentScape Companion Bridge
```

仓库可过渡期保持旧名；职责稳定后建议改为 `agentscape-companion` 或 `agentscape-desktop`。

## ADR-P05：`modal-gen-client` 不作为长期必需 daemon

有价值内容拆为 Generation Provider contract、schema、normalization、idempotency 与 fixtures；长期不保留多层 daemon 中转。

## ADR-P06：`modal-2D-client` 不再发展成第二个完整桌面产品

2D Prompt/Gallery/Batch 等用户能力迁入 Local Companion 的 2D Studio。

## ADR-P07：Artifact Identity 与 Artifact Location 分离

同一 Artifact 可以同时存在于 local companion、cloud object store、provider temporary store、compiled asset store。Artifact 身份不等于 URL、路径或 SHA-256。

## ADR-P08：Artifact / Asset / Instance 分离

```text
Artifact = 文件/字节/生成结果
Asset    = AgentScape 能理解和使用的语义资产
Instance = Asset 在某个 World 中的一次实例
```

## ADR-P09：Agent 先搜索，再生成

```text
search existing assets
→ rank / inspect
→ reuse if admissible
→ only generate missing assets
```

## ADR-P10：Companion 主动连接 Cloud

推荐 `Local Companion ──outbound WSS──► AgentScape Companion Gateway`，不把 `Browser → localhost` 作为长期云端产品架构。

---

# 3. 最终系统总图

```text
┌──────────────────────────── USER DEVICE ─────────────────────────────┐
│ Browser                                                             │
│ ┌────────────────────────────────────────────────────────────────┐ │
│ │ AgentScape Studio: Chat / World / Assets / Jobs                │ │
│ └──────────────────────────────┬─────────────────────────────────┘ │
│                                │ HTTPS / WS                         │
│ Local Companion                │                                    │
│ ┌──────────────────────────────┴─────────────────────────────────┐ │
│ │ Library / Projects / Preprocess / Viewer / Cache               │ │
│ │ 2D Studio / 3D Studio / Provider Consumer / Companion Bridge   │ │
│ └──────────┬────────────────────┬─────────────────────────────────┘ │
│            │                    │                                   │
└────────────┼────────────────────┼───────────────────────────────────┘
             ▼                    ▼
       ┌──────────┐         ┌──────────┐
       │ modal-2D │         │ modal-3D │
       │ Provider │         │ Provider │
       └────▲─────┘         └────▲─────┘
            │                    │
            └──────────┬─────────┘
                       │ same Provider Contract
                       ▼
┌──────────────────── AgentScape Cloud ───────────────────────────────┐
│ Generation Service                                                 │
│      │                                                             │
│ Agent Runtime ─ Skills ─ Asset Catalog ─ Artifact Registry         │
│      │                         │                                   │
│      ▼                         ▼                                   │
│ Planner → World IR → Compiler → World Runtime → Verification       │
│                                                                    │
│ Companion Gateway ◄──── outbound WSS from Local Companion          │
└────────────────────────────────────────────────────────────────────┘
```

---

# 4. 仓库边界重新定义

| 仓库 | 长期职责 | 应保留 | 应移出/停止扩展 |
|---|---|---|---|
| `AgentScape` | Cloud + Studio + Agent-native World 产品 | Frontend、Agent、Skills、Generation Service、Asset Catalog、Companion Gateway、Compiler、World IR、Runtime、Verification | Provider 实际推理、大模型 Secret 下发浏览器、任意本机文件访问 |
| `modal-2D` | 云端 2D Generation Provider | Text→Image Worker、Capability、Job、Artifact、Provider auth | 用户 Project、桌面 UI、AgentScape World 逻辑 |
| `modal-3D` | 云端 3D Generation Provider | Image→3D、cloud preprocess/SAM、模型 Worker、Capability、Job、Artifact | 用户本机 Project、桌面 UI、AgentScape Scene |
| `modal-3D-client` | Local Companion 过渡仓 | Local Library、Projects、Preprocess、Component selection、Canonical image、Cache、Viewer、Provider SDK、Companion bridge | 内嵌“统一 Connector 控制面”、作为 AgentScape 必经代理 |
| `modal-2D-client` | 过渡 2D UI/迁移源 | Prompt/Gallery/Batch 产品能力迁移 | 第二套长期 Tauri/DB/cache/vault |
| `modal-gen-client` | 迁移期 contract/reference source | schemas、normalization、fixtures、adapter 经验 | 长期独立中央 daemon、长期业务 Job 真值 |
| `modal-build` | Heavy workflow/build provider 工程 | EmbodiedGen build/runtime、阶段化 Provider | Local UI、AgentScape world truth |
| `EmbodiedGen` | 上游只读能力来源 | 固定版本、只读参考 | 产品控制面 |
| `AgentScape-plan` | 跨仓架构与执行权威 | ADR、Gate、契约、迁移状态 | 业务实现 |


## 4.1 当前代码事实与重构起点（2026-08-27）

本设计不是从空白画图，直接基于当前 workspace：

### AgentScape

当前 `main@812e59c` 已有：

```text
src/generation/GenerationOrchestrator.js
src/generation/GenerationJobCenter.js
src/artifacts/ArtifactDescriptor.js
src/artifacts/ArtifactRegistry.js
src/skills/registerCoreSkills.js
```

其中已经存在 provider/capability、generation jobs、Artifact import、Artifact location、verified artifact → compiler 的基础。因此目标是把现有 Generation Support 从“本地 Connector session 假设”迁成 `AgentScape Cloud GenerationService → Provider`，不是重写生成系统。

### modal-2D

当前 `main@c929a03` 已经是刻意保持很小的 SANA-Sprint Provider：固定 1024×1024 lossless PNG、SHA-256、opaque artifact id；README 明确“不包含 Web UI、SQLite、用户账号、Connector、业务编排”。**这个边界基本就是长期目标，应保留。**

### modal-2D-client

当前 `main@dd21b37` 是无桌面 UI 的本地薄 Provider Agent，已有 Session、Job mirror、Artifact cache、loopback `/v1/*`。它原计划未来并入 `modal-gen-client`；新方案改为：可复用的 2D 本地消费/缓存/UX 能力迁入 Local Companion，不再构建第二套长期客户端。

### modal-3D

当前 `master@6332d6e` 已经承担多模型 Image→3D 与 cloud preprocessing/SAM 能力。长期继续保持 Provider 身份，并与 `modal-2D` 对齐共享 Generation Provider Contract。

### modal-3D-client

当前 `master@2c6e7b5` 已经拥有 React/Tauri、Python Agent、本地 preprocessing、multi-object foreground selection、cloud generation、local project workspace、credential 与 Unified Connector 实现。**这些事实说明它已经天然接近 Local Companion**；需要拆掉的是中央 Connector 身份，不是 Project/Preprocess/Library/Viewer 这些用户价值。

### modal-gen-client

当前 `main@058ccc6` 已实现 `/connector/v1/*`、Session/Pairing、Capability Registry、Unified Job/Artifact、`modal-2D`/`modal-3D` adapters；其 adapter 通过两个 Provider Agent 的 loopback HTTP 工作。新架构将它视为**高价值 contract/reference implementation**，但不再将这个 daemon 放进长期生产必经链。

因此迁移原则是：

```text
保留已验证代码事实
        ↓
拆职责，不大爆炸重写
        ↓
Provider contract 下沉
        ↓
AgentScape direct consumer
+ Local Companion direct consumer
        ↓
Asset Catalog 汇合
```

---

# 5. `modal-2D` / `modal-3D`：统一 Provider 层

## 5.1 Provider Contract

最小逻辑资源：

```http
GET    /v1/capabilities
POST   /v1/jobs
GET    /v1/jobs/{job_id}
DELETE /v1/jobs/{job_id}
GET    /v1/jobs/{job_id}/artifacts
GET    /v1/artifacts/{artifact_id}
```

如果 Modal 原生 RPC 更适合内部调用，可以保留 RPC，但必须有同等 contract adapter。

## 5.2 Capability

```json
{
  "provider": "modal-3d",
  "operation": "asset.image_to_3d.v1",
  "revision": "...",
  "inputs": {"image": {"mime": ["image/png"], "alpha": "supported"}},
  "outputs": [{"role": "primary-glb", "mime": "model/gltf-binary"}],
  "optionsSchema": {},
  "costClass": "gpu-heavy"
}
```

## 5.3 modal-2D

第一等能力：`image.text_to_image.v1`。

输出必须区分：

```text
primary-image = lossless PNG
preview       = 可压缩预览
```

3D 下游默认只消费 `primary-image`。

## 5.4 modal-3D

推荐能力族：

```text
image.segment.v1
image.canonicalize.v1
asset.image_to_3d.v1
asset.text_to_3d.v1       # 若 Provider 支持
asset.retexture.v1        # 后续
asset.affordance.v1       # 后续
```

AgentScape Cloud 在用户电脑离线时也必须能完成 Image→3D，因此必要 cloud preprocess 不能只存在本机。

## 5.5 Provider 不拥有全局业务 ID

```text
AgentScape generation job → remote.provider_job_id
Companion local job       → remote.provider_job_id
```

两个 Consumer 各自拥有 durable projection。

---

# 6. AgentScape Cloud 重新分层

AgentScape 不再只是浏览器 Runtime Demo，而是完整 AI World Studio 产品域。

```text
AgentScape/
├── frontend/studio/
├── server/
│   ├── api/
│   ├── realtime/
│   ├── agent/
│   ├── generation/
│   ├── assets/
│   ├── companion/
│   ├── projects/
│   └── auth-policy/
├── runtime/
│   ├── world-ir/
│   ├── compiler/
│   ├── world-runtime/
│   ├── interaction/
│   ├── physics/
│   └── verification/
└── contracts/
    ├── generation/
    ├── artifact/
    ├── asset/
    ├── companion/
    └── world/
```

这是责任结构，不要求一次性搬目录。

## 6.1 Agent Service

拥有 conversation、planning、skill selection、policy、tool trace、replan barrier、world proposal/revision。Agent 不直接写 Provider HTTP，而是调用 Skills。

## 6.2 Generation Service

职责：

```text
Provider discovery
capability cache
provider selection
submit / poll / cancel
cost / policy
job projection
artifact registration
lineage
```

内部：

```text
GenerationService
  ├── Modal2DProviderAdapter
  ├── Modal3DProviderAdapter
  └── EmbodiedGenProviderAdapter
```

这是现有 `GenerationOrchestrator` 应继续演化的方向。

## 6.3 Asset Catalog

统一搜索 `built-in / cloud imported / AI generated / local companion published`。Agent 只通过 Catalog 搜资产，不扫文件系统。

## 6.4 Artifact Registry

拥有 artifact identity、MIME/format/bytes、hash/integrity、producer、lineage、locations、retention。Provider 结果、Local Companion 素材、Cloud Store 都映射到同一个 Artifact 模型。

## 6.5 World Runtime 边界

本文不推翻 `07-agent-native-world-architecture-replan.md` 的核心世界架构：

```text
Intent
→ World IR
→ Asset + Interaction/Rule Compiler
→ World Runtime
→ Physics / Navigation
→ Verification / Repair
```

当前阶段可继续让浏览器 WorldRuntime 承担交互执行；Cloud 持久化 Project / World Revision / Agent state。未来如需要 headless server Runtime，再迁移 deployment；**无论部署在哪里，只允许一套 World authority contract，不复制世界规则。**

---

# 7. Agent Skills 重新定义

Agent 不应看见 `localhost`、Modal Function 名、Volume path。

## 7.1 Asset Skills

```ts
searchAssets({ query, type?, tags?, source?, limit? })
inspectAsset({ assetId })
materializeAsset({ assetId })
compileAsset({ artifactId, assetId? })
```

## 7.2 Generation Skills

```ts
generateImage({ prompt, provider?, options? })
generate3D({ sourceArtifactId, provider?, model?, options? })
generateAsset({ prompt, strategy?: "auto" | "2d-to-3d" | "direct-text-to-3d" })
getGenerationJob({ jobId })
cancelGenerationJob({ jobId })
```

## 7.3 World Skills

继续复用 `proposeWorldIR / proposeWorldRevision / runWorldPipeline / inspectWorld`。

## 7.4 默认资产策略

```text
需求
 ↓
searchAssets()
 ↓
┌───────────────┐
│ 有合适资产？   │
└───────┬───────┘
        │
   yes  │  no
   ▼    │   ▼
reuse   │ generateAsset()
        │
        └──────┐
               ▼
          compile/admit
               ▼
            World IR
```

生成是缺失资产策略，不是覆盖已有资产的默认动作。

---

# 8. Local Companion 重新定义

现 `modal-3D-client` 作为迁移基础。

## 8.1 职责

```text
Local Companion
├── Local Asset Library
├── Projects
├── Import / Thumbnail
├── Preprocess
│   ├── BiRefNet/rembg
│   ├── component detection
│   ├── user selection
│   └── canonical RGBA
├── Artifact Cache
├── 2D Studio
├── 3D Studio
├── GLB Viewer
├── Provider SDK
│   ├── modal-2D
│   └── modal-3D
└── AgentScape Companion Bridge
```

## 8.2 不拥有的职责

禁止继续扩展：

```text
全局 AgentScape Connector control plane
Agent 的全局 generation job identity
World IR / World Runtime
Cloud Asset Catalog truth
Agent conversation
```

## 8.3 本机独立生成

```text
Text → Image:
Companion UI → Provider SDK → modal-2D → PNG → local cache → library

Image → 3D:
Companion UI → source image → optional local preprocess → canonical PNG
→ Provider SDK → modal-3D → GLB → local validation → library
```

## 8.4 发布给 AgentScape

Local asset 默认只在本机。用户显式允许后，Companion 只发布 metadata，例如：

```json
{
  "assetId": "local-asset-...",
  "displayName": "Nordic Chair",
  "semanticType": "chair",
  "tags": ["wood", "nordic"],
  "artifacts": [{
    "role": "primary-glb",
    "mime": "model/gltf-binary",
    "hash": "sha256:...",
    "availability": "local-only"
  }]
}
```

不得上传本机绝对路径、未授权目录列表或任意 filesystem capability。

---

# 9. Companion ↔ AgentScape Cloud 协议

## 9.1 连接拓扑

```text
Companion
  │ outbound TLS WebSocket
  ▼
AgentScape Companion Gateway
```

不推荐长期依赖 `Cloud Browser → http://127.0.0.1:random-port`。

## 9.2 Pairing

```text
Studio 显示一次性 pairing code
       ↓
Companion 用户确认
       ↓
设备得到 scoped device credential
       ↓
建立 outbound session
```

权限至少区分：

```text
asset.metadata.publish
artifact.materialize
artifact.preview
job.local.read          # 可选
```

默认不授予：

```text
filesystem.read.any
command.exec
secret.read
```

## 9.3 Session 消息

```text
companion.hello
companion.heartbeat
asset.upsert
asset.remove
artifact.materialize.request
artifact.materialize.progress
artifact.materialize.complete
artifact.materialize.failed
```

## 9.4 Materialize

```text
Agent Skill
  ↓
materializeAsset(assetId)
  ↓
Asset Service
  ↓
Companion Gateway
  ↓
Local Companion
  ↓
读取已授权 artifact bytes
  ↓
stream/upload
  ↓
Cloud Artifact Gate
  ├── size
  ├── MIME
  ├── hash
  ├── structure
  └── policy
  ↓
Cloud Object Store
  ↓
Artifact.locations += cloud-store
```

---

# 10. Asset / Artifact / Instance 模型

```text
Artifact（文件/bytes）
   │ compile + semantic metadata
   ▼
Asset（可复用语义资产）
   │ spawn
   ▼
Instance（World 中实例）
```

## 10.1 Artifact

```json
{
  "id": "artifact-opaque-id",
  "role": "primary-glb",
  "mime": "model/gltf-binary",
  "format": "glb",
  "hash": "sha256:...",
  "bytes": 1234567,
  "producer": {},
  "lineage": [],
  "locations": [{
    "kind": "local-companion",
    "state": "available",
    "deviceId": "device-opaque-id"
  }]
}
```

## 10.2 Asset

```json
{
  "id": "asset-nordic-chair-01",
  "displayName": "Nordic Chair",
  "semanticType": "chair",
  "tags": ["wood", "nordic"],
  "artifactId": "artifact-opaque-id",
  "bounds": {},
  "collider": {},
  "capabilities": ["sit-support"],
  "admission": "ready"
}
```

## 10.3 Instance

```json
{
  "id": "chair-7",
  "assetId": "asset-nordic-chair-01",
  "transform": {},
  "state": {},
  "relations": []
}
```

---

# 11. AgentScape Studio Frontend

## 11.1 定位

它不是 Provider Dashboard，而是：

> **AI World Studio：用户用自然语言、素材和直接编辑共同构建世界的总控制面。**

## 11.2 推荐布局

```text
┌────────────────────────────────────────────────────────────────────────┐
│ AgentScape Studio                 Project ▼       Cloud ●   Local ●    │
├───────────────┬───────────────────────────────────┬────────────────────┤
│ WORLD / ASSET │                                   │ AI / PLAN          │
│ Scene Tree    │                                   │ User: 做咖啡厅     │
│ ├ room        │                                   │                    │
│ ├ table       │            3D WORLD               │ Agent:             │
│ └ chair       │                                   │ 已找到 7 个资产     │
│               │                                   │ 缺少咖啡机和吧台    │
│ Assets        │                                   │ 正在生成...        │
│ ├ All         │                                   │ Plan / Tool Trace  │
│ ├ My Local    │                                   │                    │
│ ├ Cloud       │                                   │                    │
│ └ Generated   │                                   │                    │
├───────────────┴───────────────────────────────────┴────────────────────┤
│ Jobs / Validation / Artifact Import / Compile / World Acceptance      │
└────────────────────────────────────────────────────────────────────────┘
```

## 11.3 四个核心 Surface

### A. World

3D viewport、scene tree、selection、transform、inspector、validation findings、world revision/history。

### B. Agent

Chat、plan、tool calls、pending confirmation、fresh-replan、rejected reason、retry/repair explanation。

### C. Assets

统一视图：

```text
All / My Local / Cloud / Generated
Needs Materialization / Needs Compile / Rejected
```

支持 search、semantic filter、preview、source/location、“在世界中使用”、“生成变体”、“上传到 Cloud”、“仅保留本机”。

### D. Generation Jobs

显示 intent、provider、operation、stage、progress、cost class、parent/child、artifacts、lineage、compile status、world usage。

---

# 12. 完整业务流 1：AI 自动生成场景

用户：`“使用我已有家具做一个现代咖啡厅，缺什么再生成。”`

```text
User
 ↓
AgentScape Studio
 ↓
Agent
 ↓
World Requirement Planning
 ↓
searchAssets(requirements)
 ↓
Asset Catalog
 ├── built-in
 ├── cloud
 ├── generated
 └── local companion metadata
 ↓
Rank + Inspect
 ↓
chair        found local
table        found local
lamp         found cloud
coffee maker MISS
counter      MISS
 ↓
Materialize selected local assets
 ↓
Generate only missing assets
 ↓
modal-2D → primary PNG
 ↓
modal-3D → primary GLB
 ↓
Artifact Registry
 ↓
Asset Compiler / Admission
 ↓
World IR
 ↓
Layout / Relation / Behavior / Physics
 ↓
Runtime
 ↓
Verification
 ↓
ready / provisional / rejected
```

---

# 13. 完整业务流 2：用户本机上传素材，AI 后续选择

```text
User drops chair.glb into Companion
 ↓
Local Artifact Import
 ├── hash
 ├── format validation
 ├── thumbnail
 └── metadata
 ↓
Local Asset
 ↓
User chooses “Publish to AgentScape Catalog”
 ↓
metadata only
 ↓
AgentScape Asset Catalog
 ↓
Agent search 可以发现
 ↓
Agent chooses this asset
 ↓
materialize
 ↓
Companion uploads exact bytes
 ↓
Cloud Artifact verification
 ↓
Compiler / Admission
 ↓
World
```

默认不预先上传全部素材。

---

# 14. 完整业务流 3：用户本机生图 → 生 3D

```text
Local Companion
 ↓
Prompt
 ↓
modal-2D
 ↓
lossless PNG
 ↓
Local Artifact Cache
 ↓
(optional local preprocess)
 ↓
canonical PNG
 ↓
modal-3D
 ↓
GLB
 ↓
Local validation / viewer
 ↓
Local Library
 ↓
optional publish metadata to AgentScape
```

整个流程不依赖 AgentScape Cloud Agent。

---

# 15. 完整业务流 4：AgentScape Cloud 自己生图 → 生 3D

```text
Agent generateAsset(prompt)
 ↓
Generation Service
 ↓
modal-2D
 ↓
lossless primary image
 ↓
Artifact Gate
 ↓
modal-3D cloud preprocess/canonicalize if required
 ↓
modal-3D image_to_3d
 ↓
GLB
 ↓
Artifact Gate
 ↓
Asset Compiler
 ↓
Admission
 ↓
Asset Catalog
 ↓
World Pipeline
```

这条链完全不要求用户电脑在线。

---

# 16. Shared Generation Contract 放在哪里

需要一份**中立、版本化、跨语言**契约。

推荐形式：

```text
generation-contracts/
├── openapi/provider-v1.yaml
├── schema/
│   ├── capability.schema.json
│   ├── request.schema.json
│   ├── job.schema.json
│   ├── artifact.schema.json
│   └── error.schema.json
├── fixtures/
└── compatibility/
```

物理上可接受：

1. 新建中立仓库；
2. 暂时从 `modal-gen-client` 抽取并将其改造成 contract/reference repo；
3. 暂放 `AgentScape/contracts/generation`，但发布独立 versioned schema artifact。

推荐 **2 → 1**：先复用 `modal-gen-client` 已有 contract 经验，API 稳定后再决定是否独立仓库。不要复制 Python/JS 两套手写 schema。

---

# 17. `modal-gen-client` 迁移方案

## Phase MG-0：Freeze

停止增加新的产品 UI、世界逻辑或必须中转能力。

## Phase MG-1：Inventory

现有模块分类：

```text
KEEP-AS-CONTRACT
KEEP-AS-FIXTURE
MOVE-TO-AGENTSCAPE
MOVE-TO-COMPANION
DELETE-AFTER-MIGRATION
```

## Phase MG-2：Contract Extraction

抽出 capability、request envelope、job、artifact、error、idempotency、provider compatibility fixtures。

## Phase MG-3：AgentScape Direct Provider

AgentScape `GenerationService` 直接使用 Provider adapter。

## Phase MG-4：Companion Direct Provider

Local Companion 使用同一 Provider contract。

## Phase MG-5：Remove Runtime Dependency

验收：关闭 `modal-gen-client` daemon 后，AgentScape 和 Companion 仍都可以生图、生 3D。通过后才允许归档/重命名该仓库。

---

# 18. `modal-3D-client` 迁移方案

当前已有大量应保留代码：React/Tauri、Python Agent、Projects、Library、Preprocess、BiRefNet/rembg、Component Selection、Canonicalization、Generation history、GLB Viewer、Artifact cache、SQLite。

应删除/重构的是**职责**，不是大爆炸重写客户端。

## LC-1：内部模块重命名

从：

```text
agent/connector/*
```

逐步拆为：

```text
companion/provider/*
companion/library/*
companion/projects/*
companion/preprocess/*
companion/cache/*
companion/agentscape/*
```

## LC-2：Provider Adapter 去中心化

保留调用 `modal-2D / modal-3D`；删除“我是所有消费者的统一 Connector”假设。

## LC-3：2D UI 合入

把 `modal-2D-client` 有价值 UX 合并成 Companion 的 `2D Studio`，不运行第二个完整客户端。

## LC-4：Local Library 第一等公民

Project 是工作流容器；Library 是长期用户资产。必须支持 import、search、tag、favorite、preview、source lineage、publish/unpublish metadata、materialize state。

## LC-5：Companion Cloud Bridge

加入 outbound session，不让浏览器扫描 localhost。

---

# 19. AgentScape Frontend 迁移方案

当前 AgentScape 已有 3D viewport、任务/生成 UI、Generation Job Center 等基础，不从零重写 Renderer。

## FE-1：Shell

建立 `World | Asset Library | AI | Jobs` 四区布局。

## FE-2：Agent Console

把“任务输入”升级为 conversation、plan、tool events、approvals、world revision、failure/replan。

## FE-3：Asset Browser

连接 AgentScape Asset Catalog。

## FE-4：Companion Status

显示：

```text
Local Companion ● online
Published assets: N
Materialization queue: N
```

## FE-5：Generation UX

用户既可手工 generate image/3D，也可交给 Agent 自动决定；两种方式必须进入同一个 Generation Job / Artifact / Asset 系统。

---

# 20. Security Boundary

## 20.1 Browser

不得持久化 Modal token、Provider API secret、Companion filesystem path、Companion device private credential。

## 20.2 AgentScape Cloud

保存云 Provider credential，但只允许 Generation Service 使用。Agent Tool 输入不能直接传任意 endpoint/header。

## 20.3 Local Companion

保存本机用户 Provider credential 时使用 OS credential store；React 不读取 Secret；provider request 由 native/sidecar 发起。

未来若改成 AgentScape account 代理计费，可允许 Companion 调 AgentScape Cloud generation API；这是另一种部署模式，不能破坏 Provider/Consumer 分层。

## 20.4 Local File

Cloud Agent 永远不能发送：

```text
read("C:\\Users\\...")
```

只能发送：

```text
materializeArtifact(opaqueArtifactId)
```

Companion 自己验证 artifact 是否属于已授权 library。

---

# 21. Identity 与状态所有权

| 实体 | 权威 owner |
|---|---|
| Provider capability | 对应 Provider |
| Provider remote job | 对应 Provider |
| AgentScape generation job | AgentScape Cloud |
| Companion local job | Local Companion |
| Cloud Artifact descriptor | AgentScape Artifact Registry |
| Local Artifact bytes | Local Companion |
| Cloud Artifact bytes | Cloud Object Store |
| Asset semantic metadata | AgentScape Asset Catalog；local-only 时 Companion 保留本地源记录 |
| World IR / Revision | AgentScape |
| Runtime truth | AgentScape World Runtime authority |
| User local filesystem | Local Companion only |

不要制造跨所有系统的万能 Job ID。保留明确映射：

```text
business job id
  → provider
  → remote provider job id
```

---

# 22. API 边界建议

## 22.1 Studio → AgentScape Cloud

```http
POST /api/agent/messages
GET  /api/projects/{id}/world
GET  /api/assets
POST /api/assets/{id}/materialize
POST /api/generation/jobs
GET  /api/generation/jobs/{id}
GET  /api/companions
```

Realtime：

```text
agent.event
world.revision
world.acceptance
generation.job.updated
artifact.imported
asset.updated
companion.status
```

## 22.2 AgentScape Cloud → Provider

只使用 Generation Provider Contract。

## 22.3 Companion → Provider

只使用同一 Generation Provider Contract。

## 22.4 Companion → AgentScape Cloud

只使用 Companion Contract。不要混成一个万能 API。

---

# 23. 目录/代码 ownership 建议

## AgentScape

```text
src/
├── agent/
├── skills/
├── generation/          # Provider orchestration, not provider inference
├── artifacts/
├── assets/
├── companion/           # cloud-side companion domain
├── pipeline/
├── compiler/
├── runtime/
├── validation/
└── ui/ or frontend/     # 逐步整理 Studio
```

## modal-3D-client → Local Companion

```text
src/                     # React/Tauri UI
agent/                    # 过渡名称
  ├── library/
  ├── projects/
  ├── preprocess/
  ├── generation/
  ├── providers/
  ├── cache/
  └── agentscape/
```

`agent/connector/` 旧代码逐个迁出，不允许一次大爆炸重写。

## modal-2D / modal-3D

```text
provider/
├── capabilities
├── jobs
├── artifacts
├── workers
└── api
```

只保持 Provider 领域。

---

# 24. 实施 Gate

## Gate PA-0 — Architecture Freeze

必须完成：

- 本文成为产品拓扑权威；
- 旧“单一 Local Connector 是 AgentScape 必经路径”标记为 superseded；
- 不再新增 connector-in-connector 设计。

DoD：所有新任务能明确归属 `Provider / AgentScape / Companion / Shared Contract`。

## Gate PA-1 — Generation Provider Contract v1

完成 capability/request/job/artifact/error schema、idempotency 与 compatibility fixtures。

DoD：`modal-2D` 和 `modal-3D` 用同一个 contract test suite。

## Gate PA-2 — Direct Dual Consumer

完成两条独立 E2E：

```text
AgentScape → modal-2D/modal-3D
Companion  → modal-2D/modal-3D
```

DoD：关闭 `modal-gen-client` 仍可完成两个消费者的生成任务。

## Gate PA-3 — Local Companion Refactor

完成 local library、2D Studio、3D Studio、local preprocess、cache、provider SDK，并删除中央 Connector 身份假设。

## Gate PA-4 — Asset Catalog v1

完成：

- AgentScape searchable asset catalog；
- Artifact / Asset / Instance 明确分层；
- source/location；
- cloud/generated/local metadata。

## Gate PA-5 — Companion Bridge

完成 device pairing、outbound session、metadata publish、materialize、revoke、offline semantics。

## Gate PA-6 — AgentScape Studio v1

完成：

```text
Chat / Agent
World
Assets
Jobs
Companion status
```

## Gate PA-7 — Asset-first Agent Planning

Agent 可以：

```text
search
select
materialize
only-generate-missing
compile
compose
verify
```

## Gate PA-8 — End-to-End World Studio

验收场景：

```text
用户本机有 chair/table
Cloud Catalog 有 lamp
coffee-machine 不存在

用户说：做咖啡厅

Agent：
1. 找到 chair/table/lamp
2. materialize 本地 chair/table
3. 只生成 coffee-machine
4. compile 全部资产
5. compose world
6. verify
7. Studio 展示结果和每一步 lineage
```

---

# 25. 测试矩阵

## Provider Contract

```text
modal-2D capability
modal-3D capability
submit / poll / cancel / artifact
invalid input
idempotency
terminal replay
```

## Dual Consumer

```text
AgentScape direct modal-2D
AgentScape direct modal-3D
Companion direct modal-2D
Companion direct modal-3D
```

## Companion

```text
import local image / GLB
preprocess
restart/recovery
publish metadata / unpublish
materialize
offline / reconnect
revoke device
```

## Asset

```text
Artifact != Asset != Instance
local-only location
cloud location
same artifact multi-location
hash mismatch
MIME mismatch
GLB structural rejection
```

## Agent

```text
search-before-generate
do-not-generate-when-good-asset-exists
generate-only-missing
no silent duplicate charge
fresh replan after world mutation
rejected generated asset does not become world truth
```

## Studio

```text
Agent event streaming
World revision
Asset search
Local asset visibility
Generation jobs
Companion online/offline
Materialization progress
Validation findings
```

---

# 26. 旧计划迁移矩阵

| 旧决策 | 新状态 |
|---|---|
| “最终一个 Local Modal Connector 统一 AgentScape 所有 Provider” | **废止** |
| “AgentScape 必须先配对本地 Connector 才能生成” | **废止** |
| “modal-2D-client 与 modal-3D-client 合成中央 Connector 产品” | **废止其中央 Connector 定位** |
| “用户本机最终只有一个桌面产品” | **保留**，但产品定义改为 Local Companion |
| “Provider capability discovery” | **保留** |
| “Job/Artifact contract” | **保留并下沉为 Shared Provider Contract** |
| “Artifact integrity + lineage” | **保留** |
| “AgentScape Compiler/Admission 是资产真值门” | **保留** |
| “World IR/Runtime/Verification 是世界真值门” | **保留** |
| “浏览器不持有 Provider Secret” | **保留** |
| “EmbodiedGen bundle 不直接成为 World truth” | **保留** |

---

# 27. 立即执行的第一批任务

## ARCH-01 — 文档权威切换

修改 README、master roadmap、旧 client-unification plan 与 AgentScape 07，明确本文覆盖旧 Connector topology。

## CONTRACT-01 — Provider Contract Inventory

从以下当前实现抽取交集与冲突，形成 v1 schema：

```text
modal-gen-client
modal-3D-client/agent/connector
modal-2D
modal-3D
AgentScape/src/generation
```

## AS-GEN-01 — AgentScape Direct modal-2D

让 AgentScape Generation Service 在无 Local Companion、无 `modal-gen-client` daemon 情况下完成 Text→Image。

## AS-GEN-02 — AgentScape Direct modal-3D

完成 Image Artifact → GLB。

## LC-01 — Companion Provider Boundary

把 `modal-3D-client` 当前 connector provider 调用抽到中性 provider client；去除中央 Connector 身份假设。

## LC-02 — Local Library

把现有 Project/Artifact 能力提升为长期 Library。

## AS-ASSET-01 — Asset Catalog

统一 `built-in / generated / cloud / local-published`。

## COMP-01 — Companion Pairing + Metadata

先只做 connect、heartbeat、publish metadata、unpublish，不急着传大文件。

## COMP-02 — Materialize

再做按需 bytes 上传与 integrity gate。

## FE-01 — Studio Asset Panel

让 AgentScape Frontend 能看到 Asset Catalog 与 Local Companion 来源。

## AGENT-01 — Search-first Asset Skill

Agent 在生成前调用 `searchAssets()`。

---

# 28. 并行开发 Ownership

```text
Track A — Provider Contract
  modal-2D / modal-3D / schema

Track B — AgentScape Generation
  direct provider adapters / jobs / artifacts

Track C — Local Companion
  library / preprocess / provider SDK / desktop UX

Track D — Companion Bridge
  pairing / WSS / metadata / materialize

Track E — AgentScape Studio
  chat / world / assets / jobs

Track F — Agent Planning
  search-first / missing-asset generation / world composition

Track G — World Core
  继续 07 的 World IR / Runtime / Physics / Verification 主线
```

冲突原则：

- Track A 不修改 World core；
- Track C 不建立 Cloud world truth；
- Track D 不实现 Provider adapter；
- Track E 不直连 Provider；
- Track F 不绕过 Asset Compiler/Admission；
- Track G 不关心素材来自 Local 还是 AI generated。

---

# 29. Definition of Done

## Provider 独立

```text
AgentScape 可直接调用 modal-2D / modal-3D
Companion 可直接调用 modal-2D / modal-3D
```

## Local Companion 独立

没有 AgentScape 时，用户仍可：

```text
import → preprocess → generate → preview → library
```

## Cloud Agent 独立

用户电脑离线时，AgentScape 仍可：

```text
generate image → generate 3D → compile → build world
```

## Local Asset 可被 Agent 使用

用户电脑在线时，Agent 可以：

```text
search local metadata
→ choose
→ materialize on demand
→ compile
→ use in world
```

## Frontend 成为真正 Studio

用户可以在一个 Web Studio 中：

```text
命令 AI
看 AI 拆解
看生成任务
看/挑素材
看 Local Companion
编辑 3D World
看 Verification
```

## 没有 Connector 套娃

最终主链中不存在：

```text
AgentScape
→ modal-gen-client daemon
→ modal-3D-client connector
→ modal-3D
```

## World truth 不被生成系统污染

始终保持：

```text
Generated Artifact
≠ Asset Ready
≠ World Ready

Provider Success
→ Artifact Verify
→ Asset Compile/Admission
→ World Compile
→ Runtime
→ Verification
→ World Ready
```

---

# 30. 最终心智模型

```text
modal-2D / modal-3D
    = “我会生成”

Local Companion
    = “我拥有并管理用户本机的东西，我也能调用生成能力”

AgentScape Asset Catalog
    = “我知道现在有哪些东西可供 Agent 使用”

AgentScape Generation Service
    = “我代表 Cloud Agent 调用生成 Provider”

AgentScape Agent
    = “我决定应该用什么、缺什么、生成什么、如何拆解任务”

AgentScape World Runtime
    = “我决定这些资产和行为是否真的组成一个有效世界”

AgentScape Studio
    = “用户支配 AI、素材、生成任务和世界的总控制面”
```

这是后续所有仓库重构、API 设计和 UI 决策的默认架构。
