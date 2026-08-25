# 总体实施路线：Modal 2D/3D 创作 → EmbodiedGen 工作流 → AgentScape 世界

## 1. 目标定义

最终目标不是简单地把几个 HTTP 地址接在一起，而是形成一条可恢复、可验证、可扩展的本地/云端创作系统：

1. 将 `kaggle-inference-hub` 中的 SANA 与 Z-Image 迁成 `modal-build + modal-2d + modal-2d-client`，并保留 Prompt、batch、Gallery 与历史能力。
2. 用户在本地提交自己的图片或已抠图 RGBA，选择 Modal 上的 3D 模型，得到可在本地预览、保存和再次使用的 GLB。
3. 将 `modal-2d-client` 与 `modal-3D-client` 彻底统一为一个 Host、Connector、凭据、Job DB、Artifact Cache 和项目系统。
4. `modal-build` 在不修改 `EmbodiedGen` 的前提下，将其能力拆成可独立部署、可恢复、可组合的 Modal 阶段。
5. AgentScape 通过同一个本地安全 Connector 使用两条显式 Text→3D 方案：组合式 `modal-2d→SAM→modal-3D`，以及 EmbodiedGen Text→3D bundle。
6. 所有生成结果先进入 AgentScape 的 Asset Compiler 与 Admission；长期主链再进入 World IR、Interaction/Rule Compilation、World Runtime、Physics Capability、Navigation 与 Verification。上游模型、Planner 或具体 solver 都不能绕过验证成为未审查的“真实能力”。

本文把“先实现自己的 3D 模型转换”解释为第一条可交付纵向链：

```text
用户自有图片 / Canonical RGBA
→ 本地 modal-3D-client
→ Modal SAM 预处理（可选）
→ Modal 3D Worker
→ GLB
→ 本地校验、预览、保存
```

新增的文本创作纵向链为：

```text
Prompt → modal-2d lossless image → 可选 SAM/canonical RGBA
       → modal-3D → GLB → AgentScape Compiler

Prompt → EmbodiedGen Text→3D workflow → sim-ready evidence bundle
       → AgentScape Compiler
```

已有 3D 文件的 OBJ/FBX/STL/PLY 格式互转不作为第一里程碑；它在后续“资产导入与 AgentScape 编译”阶段按真实需求加入，避免把最初目标稀释成通用 DCC 转换器。

## 2. 不可打破的仓库边界

| 仓库 | 所有权与职责 | 允许修改 | 明确禁止 |
|---|---|---|---|
| `kaggle-inference-hub` | 2D/3D Kaggle 原型、迁移行为与历史数据基线 | 迁移期安全修正、导出/归档文档 | 继续吸收新的正式 Modal 能力；把硬编码 token/Tunnel 带入新系统 |
| `modal-2d`（新） | SANA、Z-Image 等轻量独立 Modal 2D Worker 与 Gateway | Worker、Capability、Artifact、测试、部署 | 放 Prompt AI Secret、本地 UI、Kaggle worker claim |
| `modal-2d-client`（新） | 2D Studio 过渡产品模块 | Prompt、batch、Gallery、2D Provider、legacy import | 长期复制第二套 Tauri/凭据/Job DB/Artifact Cache |
| `modal-3D` | 轻量、模型无关的 Modal 3D Worker 与预处理事实源 | Worker、Gateway、能力清单、结果契约、测试、部署文档 | 放本地 UI、持久化用户凭据、吸收 EmbodiedGen 全部逻辑 |
| `modal-3D-client` | Windows/Tauri 本地产品、Modal 凭据、任务与本地产物 | React、Tauri、Python Agent、本地 DB、Connector 协议 | 在浏览器/React 中持久化 Modal Secret；把大型模型打包进安装器 |
| `modal-client`（最终逻辑产品） | 单一 2D/3D/EmbodiedGen 桌面 Host 与 Connector | 统一 UI、Provider、Job、Artifact、Project、AgentScape 配对 | 同时运行两个 sidecar/vault/DB；以跨端口文件复制冒充统一 |
| `modal-build` | 构建 2D/3D CUDA 产物、消费固定上游版本、部署 EmbodiedGen Modal 工作流 | Builder、Release、Runtime、版本化 patch、适配层、测试 | 在付费推理容器临时编译；把权重或 Secret 发到 Release |
| `EmbodiedGen` | 上游能力与语义基线 | **不修改，只读、固定 commit/tag** | 任何业务提交、直接修补、为 Modal 改上游目录 |
| `AgentScape` | 浏览器原生 Agent-native World Compiler & Runtime；拥有 World IR、资产/行为编译、运行时与验证边界 | Provider/Connector 支撑、Compiler、World IR、Interaction/Rule、PhysicsBackend、UI、测试 | 在浏览器保存 Modal Token；把 Provider/Planner/solver proposal 直接当可执行或验证真值 |

如果 EmbodiedGen 需要兼容性修正，只能采用以下顺序：

1. 先用外部 wrapper、环境变量、固定入口绕开。
2. 确实无法绕开时，在 `modal-build/patches/<upstream-version>/` 保存最小 patch。
3. patch 必须记录上游 commit、目标文件哈希、原因、验证用例和失效条件。
4. `EmbodiedGen` 工作区始终保持上游原貌。

## 3. 当前真实基线

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

`AgentScape` 当前基线已经明显超出本计划最初的“同步 Generator + GLB URL Adapter”阶段。以 `main@d63cfe5`（v1.34.2）复核：

- Provider Registry、Connector scoped pairing/capability snapshot 已进入 main；
- async Generation Job projection/reconcile、Artifact identity/import integrity 已进入 main；
- verified Artifact → Asset Compiler → Admission 主链已进入 main；
- EmbodiedGen semantic evidence bridge 已进入 Compiler evidence 路径，但不会被提升成 Runtime truth；
- Agent-visible async generation 与 Generation Job Center core 已进入 main；
- 当前 `git stash list` 为空；旧 Live Map 记录的 WorldRevision/backend stash 不可继续当作可恢复实现；
- Runtime / Physics / Navigation / Interaction / Verification 仍然是 AgentScape 最成熟的事实层；
- `WorldRuntime.mutate()` exception atomicity 已由 `3a956dc` 闭合：partial throw 恢复 before snapshot，rollback failure fail-closed，snapshot failure 不泄漏 mutation owner；当前 full suite `143 files / 624 tests PASS`。

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

```text
┌──────────────────────────────── 本地设备 ────────────────────────────────┐
│                                                                          │
│  统一 modal-client（2D / 3D / Pipeline） + AgentScape                    │
│        │                                                                 │
│        ├── UI：Prompt、图片、模型/工作流、任务、Gallery、3D/世界预览     │
│        │                                                                 │
│        └── Local Modal Connector                                         │
│             ├── Windows 凭据管理器 / 内存 Modal Client                   │
│             ├── modal-2d / modal-3d / EmbodiedGen providers             │
│             ├── capability discovery + durable local jobs                │
│             ├── artifact cache / project lineage / 2D→3D handoff        │
│             ├── local Prompt Pipeline                                    │
│             └── loopback session/pairing security                        │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ Modal RPC（不从浏览器直传 Secret）
                               ▼
┌──────────────────────────────── Modal ───────────────────────────────────┐
│  modal-2d control plane                                                  │
│    ├── SANA Sprint / Z-Image-Turbo                                       │
│    └── lossless primary + preview artifact                               │
│                                                                          │
│  modal-3D control plane                                                  │
│    ├── SAM 3.1 preprocessing                                             │
│    ├── FastSAM3D++ / Hunyuan++ / Hermit++ / Pixal3D                    │
│    └── shared artifact contract                                          │
│                                                                          │
│  modal-build EmbodiedGen workflows                                       │
│    ├── asset.image_to_3d / asset.text_to_3d                             │
│    ├── asset.texture / asset.convert / asset.affordance                  │
│    ├── scene.background / scene.layout / scene.room                      │
│    └── simulation.validate（后置）                                       │
│                                                                          │
│  Shared: immutable artifacts, per-job stage state, hash, lineage         │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ versioned result/artifact bundle
                               ▼
┌──────────────────────────── AgentScape ──────────────────────────────────┐
│ Support: Provider / Connector / Job / Artifact                           │
│ 支撑层：Provider / Connector / Job / Artifact                            │
│                              ↓                                           │
│ World Planner → World IR → Asset + Interaction/Rule Compilation          │
│ 世界规划器 → 世界 IR → 资产 + 交互/规则编译                              │
│                              ↓                                           │
│ World Runtime → Physics Capability → Navigation / Interaction            │
│ 世界运行时 → 可替换物理能力层 → 导航 / 交互                              │
│                              ↓                                           │
│ Verification → Finding → Bounded Repair / IR Revision                    │
│ 验证 → 问题证据 → 有界修复 / IR 修订                                     │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.1 为什么必须有 Local Modal Connector

不让 AgentScape 直接调用 Modal Web Endpoint，原因不是偏好，而是当前边界决定的：

- AgentScape 是浏览器应用，现有原则明确不在浏览器保存模型或服务 Secret。
- EmbodiedGen ASGI 使用 proxy auth；`modal-2d`/`modal-3D` 使用私有 Modal RPC 和 Volume。
- lossless 图片、GLB 和 bundle 可能位于私有 Volume，不是天然的 CORS 公网 URL。
- 本地任务必须跨刷新/重启恢复，而浏览器同步 `fetch` 无法承担 3–30 分钟工作流。

因此 Connector 是统一的安全与可靠性边界。过渡期以 `modal-3D-client/agent` 的安全基线为源，最终收敛为一个中性 Local Connector；`modal-2d-client` 不长期另建第二套凭据、DB 和 artifact cache。

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

### Gate 3：2D/3D 客户端彻底统一

完成：

- 一个 Tauri Host、Local Connector、credential、DB、cache；
- 中性 capability/job/artifact/project contract；
- 2D/3D工作区与global Job Center；
- lossless `2D→SAM→3D` parent-child pipeline；
- 3D失败从3D stage重试，不重新生成2D；
- 一个AgentScape配对入口。

只有 Gate 3 通过，AgentScape 才依赖正式 Connector。

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

### Gate 6：AgentScape Generation Support Plane / 生成支撑层

原“双方案单资产接入”基础大部分已经进入 AgentScape main，因此 Gate 6 重新定义为产品级支撑层验收：

- real Connector process pair → capability → submit → restart/reconcile → artifact → compile E2E；
- local Job identity / idempotency / event cursor 不因重启重复计费；
- Artifact hash/lineage 与 Compiler/Admission truth 保持分层；
- 方案 A / B 继续显式选择，不静默双跑；
- AS-10B 只有在 provider 真实提供 model/workflow/optionsSchema metadata 后才进入；
- Provider success 永远不等于 Asset/World verified。

Gate 6 是 AgentScape Support Plane，不再阻塞 World IR contract 的设计工作。

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

### ADR-07：最终只有一个本地客户端基础设施

`modal-2d-client` 可以先作为独立工作区交付，但最终只保留一个 Tauri Host、Connector、credential、Job DB、Artifact Cache 和 AgentScape pairing。两个 localhost 应用互相传 URL 不算“彻底打通”。

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
