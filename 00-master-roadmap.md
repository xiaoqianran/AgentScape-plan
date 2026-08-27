# 总体实施路线：Modal 2D/3D 创作 → EmbodiedGen 工作流 → AgentScape 世界

> **产品拓扑更新（2026-08-27）**：`01-product-architecture-replan.md` 已成为 AgentScape Studio / Cloud Provider / Local Companion 的跨项目产品架构权威。本文中“单一 Local Connector 是 AgentScape 必经路径”“2D/3D 客户端统一成中央 Connector”的旧设计已被覆盖；World IR / Compiler / Runtime / Verification 的核心 Gate 仍继续有效。

## 1. 目标定义

最终目标是形成一套 **AI World Studio + Cloud Generation Providers + Local Companion**：

1. `modal-2D` 与 `modal-3D` 成为同级、纯粹、可独立调用的云端 Generation Provider；
2. AgentScape Cloud 通过 Skills / Generation Service **直接**调用 Provider，自动生图、生 3D、编译资产并构建世界；
3. `modal-3D-client` 演进为 Local Companion，用户本机也能独立调用同一批 Provider，并管理自己的素材、预处理、缓存和预览；
4. `modal-2D-client` 的 Prompt/Gallery/Batch 等产品能力最终迁入 Local Companion，不形成第二套长期桌面基础设施；
5. AgentScape Studio 成为用户总控制面：命令 AI、查看任务拆分、管理素材、观察生成 Job、编辑/查看 3D World 与 Verification；
6. Local Companion 只向 AgentScape 发布用户授权的 Asset metadata，真正使用 local-only 资产时再按需 materialize bytes；
7. `modal-gen-client` 不作为长期必经 daemon，其稳定 contract/schema/fixtures 下沉为共享 Generation Provider Contract；
8. 所有生成结果都必须经过 Artifact Gate → Asset Compiler/Admission → World Runtime/Verification，Provider success 永远不等于 World ready。

两条关键生成路径：

```text
AgentScape Cloud ───────────────┐
                               ├──► modal-2D / modal-3D ─► Artifact ─► Compiler ─► World
Local Companion ───────────────┘
```

```text
用户本机素材
→ Local Companion Library
→ publish metadata
→ AgentScape Asset Catalog
→ Agent 选择
→ on-demand materialize
→ Compiler / Admission
→ World
```

EmbodiedGen 仍通过 `modal-build` 以重型 Provider/workflow 形式提供 Text→3D、affordance 等能力，不直接成为 AgentScape World truth。

## 2. 不可打破的仓库边界

| 仓库 | 长期角色 | 允许 | 明确禁止 |
|---|---|---|---|
| `kaggle-inference-hub` | 历史原型/迁移证据 | 对照、归档、必要安全修复 | 吸收新的正式生产控制面 |
| `modal-2D` | 纯 2D Generation Provider | Worker、Capability、Job、Artifact、Provider API | Local Project、AgentScape World、桌面产品状态 |
| `modal-3D` | 纯 3D Generation Provider | cloud preprocess/SAM、Image→3D Worker、Capability、Job、Artifact | Local Library、桌面 UI、AgentScape World |
| `modal-3D-client` | **Local Companion 迁移基础** | Local Library、Project、Preprocess、Cache、Viewer、2D/3D Provider consumer、Companion Bridge | 成为 AgentScape 的必经中央 Connector |
| `modal-2D-client` | 过渡 2D 产品/迁移源 | Prompt/Gallery/Batch UX 迁移 | 长期第二套桌面 Host/DB/cache/vault |
| `modal-gen-client` | 迁移期 contract/reference source | schema、normalization、fixtures、adapter 经验 | 长期中央 daemon、全局业务真值 |
| `modal-build` | EmbodiedGen/build/heavy workflow Provider 工程 | Builder、Runtime、patch、阶段工作流 | Local UI、World truth |
| `EmbodiedGen` | 上游只读能力源 | 固定 commit/tag、只读审计 | 产品控制面 |
| `AgentScape` | **Studio + Cloud Agent + Asset + World 产品域** | Frontend、Agent/Skills、Generation Service、Asset Catalog、Companion Gateway、Compiler、Runtime、Verification | 浏览器保存 Provider Secret；绕过 Compiler/Verification |
| `AgentScape-plan` | 架构/契约/Gate 权威 | ADR、迁移与验收文档 | 业务实现 |

EmbodiedGen 如需兼容性修正，仍优先 external wrapper；无法绕开时才在 `modal-build/patches/<upstream-version>/` 保存带 base hash、原因与验证用例的最小 patch。

## 3. 历史实现基线 / 当前实施需重新复核

> 本节保留原计划形成时的实现快照，不再作为 2026-08-27 当前 checkout 的权威事实。当前仓库角色与重构起点见 `01-product-architecture-replan.md` 的 4.1；实施任务必须重新读取各仓 HEAD。

### 3.1 2D 前置组

`kaggle-inference-hub` 已有：

- SANA Sprint 1.6B 与 Z-Image-Turbo GGUF 双 T4 Notebook Worker；
- SQLite WAL、按模型队列、原子 claim、lease/retry、worker/history；
- AES-GCM 结果回传、FastAPI/WebSocket、React Prompt Studio/Gallery；
- 本地 OpenAI-compatible Prompt Pipeline；
- TripoSR 与多份 3D 研究 Notebook。

迁移边界：只有 SANA/Z-Image 与 2D 产品交互进入 `modal-2d` 栈；TripoSR/其他 3D Notebook 回到 `modal-3D/modal-build` 候选或归档。Kaggle 的共享默认 token、Tunnel、浏览器 localStorage token、AES 回传与 worker claim 不进入正式 Modal 架构。

`modal-2d`、`modal-2d-client` 当前尚不存在，必须按共享 contract 新建。Z-Image 的 T4/SM75 预编译包也必须在 `modal-build` 为目标 Modal GPU 重建，而不是假定 ABI/架构兼容。

### 3.2 3D 第一组

`modal-3D` 已有：

- `modal_3d/gateway.py`：CPU Router，按模型名定位 Worker 并 `spawn`。
- 4 个活动 Image→3D Worker：FastSAM3D++、Hunyuan2.1++、Hermit-TRELLIS2++、Pixal3D。
- `modal_3d/sam3_1.py`：concept segmentation、box refine、bit-packed masks、CPU canonical RGBA materialization。
- 统一基础结果：`model + artifact(path/bytes/mime) + timing + metrics`。
- 每个 GPU Worker 独立镜像/权重 Volume，GPU `max_containers=1`，CPU 预加载权重，离线 GPU 推理。

`modal-3D-client` 已有：

- Tauri 启动随机 loopback 端口的 Python sidecar，并生成每次启动的会话令牌。
- Windows Credential Manager 保存 Modal 凭据；Secret 不重新回传 React。
- Cloud SAM → candidate → canonical RGBA → 选择任一四模型 → async Job → GLB 下载的真实主链。
- Artifact 使用共享 `modal-3d-artifacts` Volume，输入以 SHA-256 内容寻址。
- 本地 model registry/recommended profile 与按需 Three.js GLB Viewer。
- 当前工作区中的 SQLite Job 持久化基础，可保存本地 ID、remote call、终态/result。
- 当前工作区新增项目数据库与项目式 RGB→SAM→canonical RGBA→3D UI，source、selection、canonical、Job 与结果可按 project 关联。
- 当前工作区新增硬件探测、SAM mode settings 与 `/v1/capabilities` 第一条切片；它能区分 cloud 是否连接、本地 GPU/文件是否满足，但在本地 runtime 协议完成前会正确报告 local SAM unavailable。

关键缺口：模型事实仍在 client 重复；当前 capability 只覆盖本地硬件/SAM 可用性而不是云端模型/option 事实；Job DB 与 Project DB 尚未收敛为版本化 schema；仍缺 migration、remote kind、启动 reconcile、内容寻址 artifact cache、direct RGBA 产品入口、Connector 配对协议和系统级测试。

### 3.3 第二组

`modal-build` 已有：

- EmbodiedGen v2.0.0 固定 commit 的 release-only Runtime；推理镜像没有 `nvcc`。
- Image→3D 已拆为 rembg、SAM3D、mesh/UV、lite GPU texture、CPU finalize。
- Text→3D 已用 Kolors Text→Image 复用上述主链。
- Modal Dict 保存 Job state、traffic state 与阶段 handoff；Volume 保存输入、阶段产物与结果。
- `/jobs`、`/text-jobs`、`/jobs/{id}`、文件下载与定时清理。
- autoscale profile、成本尾部估算、供应链 hash、API path/TTL 等本地测试。
- 当前工作区已出现实验性 retexture 纵向切片：固定 RoboAssetGen revision/权重 preload、L40S `RetextureWorker`、派生 Job API、离线 load、几何保持/OBJ依赖/GLB/URDF/video 校验与本地静态/契约测试。

`EmbodiedGen` 当前工作区本身还存在未提交的 Modal/headless 实验文件与上游文件修改。它们只能作为审计/迁移输入；本计划不覆盖、不删除，也不允许正式部署继续从 dirty checkout 隐式消费。可复用逻辑必须迁到 `modal-build` wrapper 或带 base hash 的版本化 patch 后再进入 Gate。

关键缺口：Runtime 仍集中在一个大文件；正式协议覆盖 Image/Text，两者之外的 retexture 仍是实验性派生端点，尚无真实 Modal GPU canary、独立 capability/部署、通用 artifact manifest、恢复/幂等证据；整体仍缺取消、阶段重试/恢复与统一 artifact index。场景、布局、房间、affordance、转换和仿真尚未 Modal 化。

### 3.4 第三组

`AgentScape` 当前基线已经明显超出本计划最初的“同步 Generator + GLB URL Adapter”阶段。以 `main@8253f7d`（v1.34.2）复核：

- Provider Registry、Connector scoped pairing/capability snapshot 已进入 main；
- async Generation Job projection/reconcile、Artifact identity/import integrity 已进入 main；
- verified Artifact → Asset Compiler → Admission 主链已进入 main；
- EmbodiedGen semantic evidence bridge 已进入 Compiler evidence 路径，但不会被提升成 Runtime truth；
- Agent-visible async generation 与 Generation Job Center core 已进入 main；
- 当前 `git stash list` 为空；旧 Live Map 记录的 WorldRevision/backend stash 不可继续当作可恢复实现；
- Runtime / Physics / Navigation / Interaction / Verification 仍然是 AgentScape 最成熟的事实层；
- `WorldRuntime.mutate()` exception atomicity 已由 `3a956dc` 闭合：partial throw 恢复 before snapshot，rollback failure fail-closed，snapshot failure 不泄漏 mutation owner；当前 full suite `150 files / 662 tests PASS`。

因此第三组未来不再以 `AS-11→AS-19` 线性推进，而使用：

> [`group-3-agentscape/07-agent-native-world-architecture-replan.md`](./group-3-agentscape/07-agent-native-world-architecture-replan.md)

新的主轴是：

```text
World Intent / 世界意图
        ↓
World IR / 世界中间表示
        ↓
Asset Compiler + Interaction/Rule Compiler
资产编译 + 交互/规则编译
        ↓
World Runtime / 世界运行时
        ↓
Verification + Repair / 验证与修复
```

Provider / Connector / Job / Artifact 从“未来主架构”调整为 **Support Plane / 支撑层**；双生成策略是 Asset Sourcing Strategy / 资产来源策略；Environment/Room 是核心编译链稳定后的内容与规模化能力。

## 4. 目标架构

完整产品拓扑以 [`01-product-architecture-replan.md`](./01-product-architecture-replan.md) 为权威。总图压缩如下：

```text
                         AgentScape Studio
                     Chat / World / Assets / Jobs
                               │
                               ▼
                        AgentScape Cloud
                  ┌────────────┼────────────┐
                  │            │            │
                  ▼            ▼            ▼
              Agent/Skills  Asset Catalog  World Core
                  │            ▲            │
                  ▼            │            ▼
          Generation Service   │       Compiler/Runtime/
             │          │      │       Verification
             ▼          ▼      │
        modal-2D     modal-3D  │
             ▲          ▲      │
             │          │      │
             └────┬─────┘      │
                  │            │
             Local Companion ──┘
        Library / 2D / 3D / Cache
              metadata + materialize
```

### 4.1 两个平级 Consumer

```text
AgentScape Cloud ──direct──► Provider
Local Companion ──direct───► Provider
```

AgentScape Cloud 不依赖用户电脑在线才拥有生成能力；Local Companion 也不依赖 AgentScape 才能管理素材和生图/生 3D。

### 4.2 Companion 不是 Connector 中转站

Local Companion 只承担用户本机域：

```text
filesystem / library / project / preprocess / local cache / viewer
```

以及与 AgentScape 的安全桥：

```text
outbound pairing/session
metadata publish
on-demand materialize
```

### 4.3 AgentScape Frontend 是总控制面

Browser 只调用 AgentScape Cloud，不直接持有 Modal Secret。用户在 Studio 中支配 AI、素材、生成任务、World 和 Verification。

### 4.4 Shared Generation Contract

`modal-2D`、`modal-3D`、AgentScape、Local Companion 共享 versioned Provider Contract：Capability / Request / Job / Artifact / Error。`modal-gen-client` 的稳定协议资产迁入这里，而不是继续保留一层必须运行的 daemon。

## 5. 执行顺序与硬依赖

### Gate 0：冻结跨项目契约

先完成：

- capability schema；
- job/envelope、状态机、错误码；
- artifact descriptor、hash、MIME、TTL；
- 坐标、单位、GLB/URDF bundle 规范；
- provider/workflow/model ID 命名；
- ready/provisional/rejected 映射。

没有 Gate 0，不允许 2D、3D、EmbodiedGen、客户端和 AgentScape 各自新增一套不兼容 API。

### Gate 1：Kaggle 2D → Modal 2D

完成：

- SANA/Z-Image exact model/release/weights pin；
- `modal-build` SM89 release与offline runtime；
- `modal-2d` 两个独立 Worker/Gateway/capability；
- lossless primary + preview artifact；
- `modal-2d-client` single/batch/Prompt/Gallery/历史恢复；
- Kaggle shadow comparison与legacy import。

TripoSR和其余3D Notebook不进入此Gate。

### Gate 2：第一组本地 Image→3D 完整闭环

完成：

- 用户自有图片与 pre-matted RGBA 两种入口；
- 4 个活动 3D 模型统一选择与 option profile；
- 可恢复 Job；
- GLB hash 校验、本地落盘、3D 预览；
- SAM 自动/云端/跳过策略；
- 真实 Windows 安装包 smoke test。

### Gate 3：Local Companion 产品化

本 Gate 已按 `01-product-architecture-replan.md` 重定义。最终用户本机仍只保留一个桌面产品，但它是 **Local Companion**，不是 AgentScape 的中央 Connector。

完成：

- 以 `modal-3D-client` 为迁移基础收敛一个 Local Library / Project / Cache / Viewer；
- 2D Studio 与 3D Studio 都直接消费共享 Generation Provider Contract；
- `modal-2D-client` 的 Prompt/Gallery/Batch 等 UX 迁入 Companion；
- Companion 可独立调用 `modal-2D`、`modal-3D`；
- Companion 通过 outbound session 向 AgentScape 发布授权资产 metadata，并支持按需 materialize；
- 不再把 AgentScape 的云端生成流量强制经过 Companion。

AgentScape 与 Companion 是 Provider 的两个平级 Consumer；Gate 3 不再是 AgentScape 调 Provider 的前置依赖。

### Gate 4：EmbodiedGen 统一工作流内核

先不扩功能，先把已有 Image/Text→3D 迁入通用工作流契约，保持结果和性能不回退：

- versioned workflow definition；
- stage state、artifact index、resume/cancel/idempotency；
- RPC 入口供 Connector 调用；
- 原 ASGI API 保持兼容或给出迁移期。

### Gate 5：高价值 EmbodiedGen 阶段

按 AgentScape 价值排序：

1. Texture Edit：先把当前实验性 retexture 纵向切片补齐生产门禁。
2. Asset Convert（尤其 URDF/MJCF/可供 AgentScape 消费的 GLB bundle）。
3. Affordance / Part evidence。
4. Scene background。
5. Layout。
6. Room。
7. 仿真、soft-body、robot-learning 工具（可选后置）。

### Gate 6：AgentScape Direct Generation + Asset Support Plane

按 `01-product-architecture-replan.md`，Gate 6 不再验收“真实 Local Connector 必经链”，而验收 AgentScape Cloud 自己作为 Provider Consumer 的生成支撑层：

- AgentScape `GenerationService` 直接发现并调用 `modal-2D` / `modal-3D` / EmbodiedGen Provider；
- Cloud generation job、provider remote job、Artifact identity 明确分层；
- Provider success → Artifact verify → Compiler/Admission，绝不直接成为 Asset/World truth；
- `searchAssets()` 默认先于 `generateAsset()`，只生成真正缺失资产；
- Local Companion 资产通过 Asset Catalog metadata + on-demand materialize 进入同一 Compiler；
- 关闭 `modal-gen-client` daemon、Local Companion 离线时，AgentScape Cloud 仍可独立完成 Text→Image→3D→Compile。

Gate 6 与 Local Companion 产品线可并行，不阻塞 World IR contract。

### Gate 7：AgentScape World Compilation Core / 世界编译核心

未来 AgentScape 核心 Gate 细分为 `07` 的 G0～G6：

1. G0 Runtime Truth & Mutation Atomicity；
2. G1 World IR vNext；
3. G2A PhysicsBackend Interface Parity；
4. G2B Interaction & Rule Contract；
5. G3 Executable Behavior Vertical Slice；
6. G4 Prompt → World IR → Canonical World Compilation；
7. G5 Semantic Asset Automation；
8. G6 World-level Acceptance + Local Repair。

完成后的核心闭环必须是：

```text
Prompt / 用户意图
    ↓
World IR / 世界 IR
    ↓
Asset + Behavior Compile / 资产+行为编译
    ↓
Physics Capability Admission / 物理能力准入
    ↓
Runtime / 运行时
    ↓
Verification / 验证
    ↓
Finding → Constrained Revision / 问题→受约束修订
```

### Gate 8：Rich Physics & Scale / 丰富物理与规模化

只有 Gate 7 核心 contract 稳定后推进：

- G7 validation-only physics backend；
- Physics Capability Router / 可替换物理能力路由；
- 必要时第二 live backend，但必须有 snapshot/restore/coupling/verification；
- Genesis、PhysX 等只作为候选 backend，不预先承诺为生产依赖；
- Environment/Room bundle；
- large-world streaming；
- offline restore；
- richer grasp/IK；
- soft-body / cloth；
- multi-agent。

AgentScape 不重新发明 solver；它负责编译 PhysicsRequirement、选择已准入能力并管理 truth authority。

## 6. 统一成功标准

### 6.1 图片级成功

- 2D Job 终态为 `succeeded`；
- lossless primary 与 preview 角色分离，bytes/SHA-256/MIME/尺寸匹配；
- prompt、模型、revision、seed、effective options 可追溯；
- primary 可离线打开，也可作为 3D downstream input；
- 3D pipeline 不会误用有损 preview；
- single/batch 与 parent-child Job 跨客户端重启恢复。

### 6.2 资产级成功

不能仅以“Modal 函数返回 200”作为成功。至少满足：

- Job 终态为 `succeeded`；
- 所有声明的必需 artifact 存在，大小与 SHA-256 匹配；
- GLB 可解析，默认 scene 非空，几何数量大于 0；
- Web budget 未超硬门；
- provenance 可追溯到 provider、model/workflow version、upstream commit、seed；
- 进入 AgentScape 后得到明确 `ready / provisional / rejected`；
- `provisional` 不得被 UI 或 Agent 表述为已验证。

### 6.3 世界级成功

- WorldSpec schema 有效、instance ID 唯一；
- 所有 required assets 完成 admission；
- 位置与关系由 Runtime 的 Physics/Spatial truth 验证；
- `WorldValidator` 无 hard finding；
- navigation rebuild 成功且关键点可达性符合预期；
- rejected 世界回滚到调用前 scene；
- scene 序列化后可恢复，并保持 asset/version/hash lineage；
- Agent Tool outcome 与 world admission 一致。

### 6.4 可靠性成功

- 本地进程、浏览器或 Connector 重启后可恢复 Job；
- 重复提交同一个 idempotency key 不重复计费/推理；
- cancel 有确定语义，无法终止的阶段标明 `cancel_requested`；
- 超时、远端输出过期、Volume artifact 缺失、模型输入不合法均有机器可读错误；
- 清理任务不删除活跃 Job，也不碰非 API/benchmark 目录。

## 7. 关键架构决策

### ADR-01：能力发现代替多处硬编码

当前模型名分别存在 Kaggle Hub config/frontend、`modal-3D/common.py`、Gateway、client Python 与 TypeScript。目标是 `modal-2d`、`modal-3D`、EmbodiedGen 各自发布 versioned capability document，由统一客户端聚合缓存但不另立事实源。

### ADR-02：本地 Job 是用户可见主身份

Modal `FunctionCall.object_id`、EmbodiedGen `job-...`、ASGI job ID 都是远端实现细节。客户端创建稳定本地 Job ID，并保存 `remote.kind + remote.id + deployment revision`，从而统一恢复和迁移。

### ADR-03：Artifact 以 descriptor/bytes 进入 AgentScape

不要求私有 Modal Volume 产物变成永久公网 URL。Connector 用会话认证下载，AgentScape 取得 bytes 后交给 Compiler，并保存到 IndexedDB compiled store。短期 URL 只作为 transport，不作为持久化身份。

### ADR-04：EmbodiedGen bundle 不直接等于 AgentScape Manifest

URDF、collision mesh、affordance JSON 是高价值 evidence，但坐标、节点、joint target 与 Rapier 契约仍需编译与验证。Adapter 只做证据映射，不越权提升可执行能力。

### ADR-05：优先模块化 App/Image，而不是一个万能镜像

EmbodiedGen 的 asset、texture、scene、room、affordance、simulation 依赖差异巨大。共用 control plane 和 artifact contract，但重资源 Worker 应按能力族拆 App/Image/weights，避免任一功能让所有冷启动变慢。

### ADR-06：Scene3D 的首版只消费 Mesh

AgentScape 当前是 Three.js GLB-first Runtime，不原生渲染 3DGS。首版 scene background 将 `mesh_model` 转成受预算约束的 GLB；3DGS PLY 作为保留 artifact。只有测得 Mesh 方案无法满足目标时，才单独立项 3DGS renderer。

### ADR-07：最终只有一个 Local Companion，但它不是中央 Connector

`modal-2d-client` 可以作为过渡工作区；最终用户本机只保留一个由 `modal-3D-client` 演进的 Local Companion，统一本机 Library、Project、Preprocess、Cache、Viewer、2D/3D Studio。**但 AgentScape Cloud 不经它中转 Provider 调用**：AgentScape Cloud 与 Local Companion 都直接消费共享 Generation Provider Contract。

### ADR-08：2D Primary 必须无损

Kaggle Hub 当前 WebP quality 90 可继续作为历史/preview，但正式 2D→3D 链默认消费 lossless primary。Preview 与 primary 是不同 artifact role，类型和 UI 都不能混用。

### ADR-09：AgentScape Text→3D 使用两条显式策略

组合式 `modal-2d→SAM→modal-3D` 与 EmbodiedGen Text→3D bundle 在 capability、Job、UI、Tool 和 lineage 中保持独立。失败跨策略 fallback 必须创建新 linked request、展示额外成本并获得 policy 允许。

## 8. 明确非目标

- 不在首版实现通用 Blender/DCC 编辑器。
- 不让浏览器直接持有 Modal、Hugging Face、Tencent 或 LLM Secret。
- 不把所有 EmbodiedGen 功能塞进一个部署里。
- 不把上游模型给出的 `open/close` 标签直接升级为 AgentScape action。
- 不在没有性能证据时提前做 streaming、KTX2、LOD 或多 GPU 并发。
- 不承诺当前阶段训练新模型。
- 不修改 `EmbodiedGen` 仓库。
- 不把 Kaggle 的默认 token、Cloudflare Tunnel、AES 上传协议或 worker claim 复制到 Modal。
- 不长期维护两套桌面 sidecar、credential、Job DB 与 cache。
- 不默认同时运行两条 AgentScape Text→3D 策略做昂贵竞赛。

## 9. 计划维护规则

AgentScape 的实时实现进度不回写成长期架构事实；HEAD/branch/stash、下一可执行切片与验证证据维护在 `group-3-agentscape/04-live-execution-map.md`。未来核心架构、G0～G8 与多 AI ownership 维护在 `group-3-agentscape/07-agent-native-world-architecture-replan.md`。具体任务使用 `05-execution-task-spec-template.md`；Provider/Artifact 稳定 contract 变化再更新 02/03 与本总路线。

实施时每个里程碑必须更新：

- 当前代码事实与计划差异；
- 完成的 contract version；
- benchmark 环境和原始结果路径；
- 尚未关闭的风险；
- 下一 Gate 的进入条件。

任何跨项目协议改动先改 `cross-cutting/01-master-contracts.md`，再改各仓库计划，避免文档再次分叉。
