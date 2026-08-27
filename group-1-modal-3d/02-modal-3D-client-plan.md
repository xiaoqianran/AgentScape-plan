# `modal-3D-client` 文件级实施计划

> **架构迁移提示（2026-08-27）**：`modal-3D-client` 的长期定位已由仓库根目录 `01-product-architecture-replan.md` 重定义为 **Local Companion**。Project/Preprocess/Library/Cache/Viewer/Provider consumer 继续保留；“向 AgentScape 提供中央 Local Modal Connector、最终并入 modal-gen-client”的旧职责废止。本文其余文件级事实和本地安全基线仍可复用。

> 执行进度、当前 HEAD、已验证证据与下一批具体任务见 [`00-execution-status.md`](./00-execution-status.md)。本文件保留为设计/验收基线。

## 1. 项目定位

`modal-3D-client` 是第一组的本地产品面，也是后续 AgentScape 访问 Modal 的安全连接基础。它必须同时承担四类责任：

1. Tauri 管理桌面应用、sidecar 生命周期和操作系统凭据。
2. Python Agent 独占 Modal Client、私有 Volume、远端 FunctionCall 与长任务恢复。
3. React 负责输入、选择、进度、预览、结果管理，不直接接触云端 Secret。
4. Local Modal Connector 向 AgentScape 提供受限、版本化、可配对的本机 API。

它不实现云端模型推理，也不把模型名、参数范围和部署名变成新的事实源。

本计划先完成第一组的 3D 产品闭环，但不把 `modal-3D-client` 固化成长期独立基础设施。客户端统一 Gate 通过后，本项目的 UI/Provider/Viewer 演进为中性 `modal-client` 中的 3D workspace；Tauri Host、Connector、vault、Job DB、Artifact Cache 与 Project Store 只能保留一份。

## 2. CodeGraph 当前事实

本计划按 2026-08-23 当前工作区快照编写。当前主链已经不是最早的单模型原型：

```text
Tauri 启动随机 127.0.0.1 端口 Python Agent
→ 从 Windows Credential Manager 直接恢复 Modal 凭据
→ Cloud SAM segment / candidate / materialize
→ 四模型 registry + recommended profile
→ modal-3D Gateway 异步 FunctionCall
→ 本地 Job 轮询
→ 私有 Volume 下载 GLB
→ Three.js Viewer / 下载
```

最新工作区还加入了第二条“项目式”产品切片：项目 source 落本地 app-data/SQLite，SAM provider selection、canonical path、generation Job 与结果回写到同一 Project；同时增加硬件探测、持久 SAM mode setting 和 `/v1/capabilities`。该 capability 当前只表达本机硬件与 SAM local/cloud 可用性，本地 SAM runtime 尚未启用，不能将它误记为已经完成的云端模型 capability discovery。

### 2.1 关键文件与成熟度

| 文件 | 当前责任 | 当前成熟度 | 计划重点 |
|---|---|---|---|
| `src-tauri/src/lib.rs` | sidecar 启停、随机 token/port、握手、日志、父进程生命周期 | 已有但需加固 | 安装升级、崩溃恢复、配对作用域、日志脱敏 |
| `src-tauri/src/credentials.rs` | Windows Credential Manager 保存/恢复 Modal token | 已有 | 凭据版本、删除验证、非 Windows 行为、权限测试 |
| `agent/main.py` | FastAPI 路由、session middleware、Modal/SAM/asset/job API | 已有但边界会扩大 | 统一 envelope、Connector scope、能力发现、错误契约 |
| `agent/capabilities.py` | 本机硬件与 local/cloud SAM 可用性切片 | 新增、局部 | 与云端 model/workflow capability 分层；版本/hash/cache；不得把“文件存在”当 runtime ready |
| `agent/settings.py` / `agent/storage.py` | SAM mode 与统一 app-data 目录 | 新增基础 | 原子设置写入、schema migration、权限/备份，并纳入最终统一客户端目录 |
| `agent/sam_provider.py` | auto/local/cloud 选择；当前 local 明确 unavailable | 新增骨架 | 完整 provider contract、health、版本、fallback policy 和恢复语义 |
| `agent/modal_client.py` | 当前会话 Modal Client | 已有 | 重连状态、身份诊断、并发安全、最小暴露 |
| `agent/artifacts.py` | SHA-256 上传、Volume 流式读取、安全相对路径 | 已有 | descriptor/hash 验证、namespace、原子本地落盘、lease |
| `agent/sam.py` | Cloud SAM RPC | 已有 | auto/local/cloud/skip provider、恢复与错误映射 |
| `agent/generation.py` | model/profile → Gateway submit | 已有 | capability 驱动、幂等 key、deployment revision |
| `agent/models.py` | 本地四模型展示与 profile 参数翻译 | 已有但重复事实 | 迁移为云端 capability cache，保留签名 fallback |
| `agent/jobs.py` | 本地 ID、远端 FunctionCall、终态、SQLite 持久化 | 正在形成 | schema migration、重启 reconcile、远端类型抽象、结果快照 |
| `agent/projects.py` | Project SQLite、source 文件、SAM/canonical/Job/result 关联 | 新增纵向切片 | 与 Job/Artifact DB 事务收敛、hash/lease、migration、direct RGBA 与导入 |
| `src/agent.ts` | React 的 typed local API | 已有 | version negotiation、typed error、SSE/增量事件、AgentScape client |
| `src/App.tsx` | 连接、项目式图片→SAM→四模型 Job、预览、下载 | 已有 MVP/项目纵向切片 | 显式状态机、完整历史、直接 RGBA、跨重启恢复 UI |
| `src/GlbViewer.tsx` | 按需 Three.js GLB 预览与资源释放 | 已有 MVP | 结构/预算提示、截图、错误详情、坐标/尺度显示 |
| `scripts/build-agent.ps1` | PyInstaller sidecar | 已有 | 可重复构建、签名/hash、升级兼容、Windows smoke |
| `scripts/smoke-agent.ps1` | sidecar smoke | 已有基础 | session、DB、凭据恢复、模拟远端、打包后 E2E |

### 2.2 当前 SQLite 能力的准确定位

当前 `agent/jobs.py` 已创建 `jobs.sqlite3` 并保存本地 ID、model、remote call ID、状态、结果和错误。这解决了“进程内 dict 完全丢失”的第一层问题，但还不能直接宣称完整恢复，原因是：

- 表没有 `schema_version`/migration 管理；
- 没有 `remote_kind`，只能恢复 `modal.FunctionCall`；
- 没有 request payload/hash/idempotency key；
- 没有 `updated_at`、stage、progress、attempt、artifact cache 状态；
- 重启后只加载记录，不主动 reconcile 非终态；
- Modal 凭据未连接时无法区分“待重连”和“远端失败”；
- `OutputExpiredError` 后若 GLB 未缓存，本地无法恢复结果；
- cancel 成功、请求已发出和远端真正停止尚未拆分。

因此计划标记为“持久化基础已有，恢复语义待完成”。

## 3. 目标本地架构

```text
React Product UI                    AgentScape Browser Runtime
        │                                      │
        ├──── Tauri invoke ─────┐               │ loopback connector
        │                       │               │ session pairing
        ▼                       ▼               ▼
Tauri Host ─────────────── Python Local Modal Connector
  lifecycle                    ├── Auth Session
  OS credential vault          ├── Capability Cache
  app data directory           ├── Durable Job Store
  process/token pairing        ├── Artifact Cache / Project Store
                               ├── modal-3D Provider
                               └── EmbodiedGen Provider（后续）
                                         │
                                         ▼
                                   Private Modal RPC
```

React 产品 UI 与 AgentScape 可以复用同一 Python 连接核心，但授权面不同：桌面主窗口可以提交/取消/下载；AgentScape 配对会话只获得用户明确授予的 provider/workflow scope。

## 4. 产品工作流

### 4.1 工作流 A：普通 RGB 图片

1. 用户选图片；本地先检查格式、bytes、像素和解码。
2. 用户选择 `SAM=auto/cloud/local`，输入 concept。
3. 展示候选框、score 与可选 refine。
4. 用户确认候选；得到 canonical RGBA descriptor。
5. 仅展示能消费该输入的模型和 profile。
6. 创建本地 Job，再远端提交；UI 不因窗口切换停止任务。
7. 终态后下载、验证 SHA-256、原子保存，才标记“本地可用”。
8. Viewer 解析成功后允许导出；解析失败保留远端结果和诊断。

### 4.2 工作流 B：用户已有透明 RGBA

1. 本地解码并确认 alpha 不是全 0 或全 255。
2. 显示前景 bounds、透明像素比例和背景边缘预览。
3. 用户可选择“直接使用”或“重新分割”。
4. 直接上传到内容寻址 namespace，跳过 SAM。
5. 后续与工作流 A 相同。

这是“先实现自己的 3D 模型转换”的必要入口；不能强迫已经准备好的 RGBA 再经过 SAM。

### 4.3 工作流 C：恢复历史任务

1. Agent 启动后打开 DB，完成 schema migration。
2. 没有 Modal 会话时，非终态显示 `connection_required`，不改写为 failed。
3. 连接后按 deployment/provider 分组 reconcile。
4. 远端仍在运行则继续轮询；已成功则拉取 result envelope。
5. 远端结果已过期但本地缓存完整则标记 `succeeded_local`；两边都无结果才是 `expired`。
6. 用户可从历史页重新预览、另存、复制参数重跑或清理。

### 4.4 工作流 D：AgentScape 请求单资产

1. AgentScape 与 Connector 完成本机配对和 capability negotiation。
2. AgentScape 提交 `modal-3d.asset.image_to_3d.v1` 或版本化 EmbodiedGen workflow 请求。
3. Connector 返回本地 Job ID，后续通过事件/轮询观察。
4. 结果以受限 artifact endpoint 流式提供，并带 hash/length/MIME。
5. AgentScape Compiler 获取 bytes 后写入自己的 IndexedDB；Connector URL 不成为资产长期身份。

## 5. 工作包

### C3D-00：工作区基线与变更保护

任务：

1. 对当前未提交的 SQLite/Tauri 改动先做行为清单，避免计划实施时覆盖用户工作。
2. 冻结一组 MVP smoke：启动、凭据恢复、Cloud SAM、四模型枚举、提交、取消、Viewer。
3. 记录 Agent、Tauri、React、Modal SDK、DB schema 的版本组合。
4. 为本地 app-data、临时 handshake/log、PyInstaller sidecar 明确路径和生命周期。
5. 建立“当前已有、部分已有、尚未实现”的证据表，并随里程碑更新。

验收：现有四模型链在任何结构调整前有可重复回归基线。

### C3D-01：云端能力发现与离线 fallback

目标是移除 `agent/models.py` 中易漂移的模型事实，同时保留云端不可达时可解释的 UI。

任务：

1. Agent 从 `modal-3D` 获取 `modal-3d.capabilities.v1`。
2. 验证 schema、contract compatibility、signature/revision 后缓存。
3. 缓存记录 `fetched_at`、deployment、ETag/hash、过期策略。
4. UI 展示模型 status：available/degraded/disabled/incompatible。
5. profile 由 capability 给出；Python 只做 schema 校验和提交映射。
6. 内置 fallback 只允许展示最后已知模型或诊断，不允许在未知兼容性下盲目提交。
7. 如果云端 schema 比客户端新且不兼容，禁用相关模型并显示升级要求。

迁移：`/v1/models` 保持一个兼容窗口，但响应来自 capability cache，而非硬编码 tuple。

### C3D-02：本地输入管线

新增纯本地输入检查层，必须在上传/占用 GPU 前完成：

- MIME magic 与扩展名一致性；
- bytes、width、height、总像素；
- EXIF orientation 规范化；
- alpha extrema/coverage；
- RGB/RGBA/color profile 规范；
- 文件 hash 和 source display name；
- 不保留原路径中的隐私信息到远端 lineage；
- 拒绝 animated/multipage 等未支持输入，或明确只取第一帧。

上传后用服务端返回 hash 对照本地 hash。canonical RGBA 的 descriptor 需要同时保存 source hash、SAM selection lineage 和输出 hash。

### C3D-03：SAM provider 策略

统一接口包含 `segment`、`refine`、`materialize`、`capabilities`，provider 可为：

| 模式 | 选择逻辑 | 首版要求 |
|---|---|---|
| `skip` | 输入已是合格 canonical RGBA | 必须通过 alpha/input validation |
| `cloud` | 显式 Cloud SAM 3.1 | 现有主链，补恢复/TTL/错误 |
| `local` | 本机能力和模型包均可用 | 后置可选下载，不进入基础安装包 |
| `auto` | 优先满足质量与可用性的 provider | 决策可见、失败可切换，不静默换模型 |

`auto` 不能只检查“有 GPU”；还需检查本地模型版本、显存、可用磁盘和 provider health。Local 与 Cloud 必须返回同一 selection/candidate/materialize 语义。

### C3D-04：模型/profile/option 产品化

UI 不直接暴露任意 Python kwargs。分两层：

1. `recommended`、`fast`、`quality` 等经过验证的 profile；不存在则只展示 available profile。
2. Advanced options 根据 JSON Schema 渲染，默认折叠并显示成本/质量影响。

要求：

- profile 与 effective options 都进入 Job history；
- seed 可编辑、可复制重跑；
- model 切换时重新验证 input requirements；
- Pixal3D 等大纹理结果显示 Web budget 风险；
- benchmark warm time 标为历史参考，不伪装为 SLA；
- capability disabled 后，历史 Job 仍可读取但不可重复提交。

### C3D-05：Durable Job Store v1

在现有 SQLite 基础上定义 migration-managed schema。逻辑实体至少包括：

**jobs**

- local job ID、kind、provider、operation、model/workflow；
- request hash、idempotency key、contract version；
- remote kind/ID/deployment revision；
- status、stage、progress、attempt、cancel state；
- created/submitted/started/updated/completed；
- normalized error code/message/retryability；
- result envelope、primary artifact ID；
- owner session/project/source application。

**artifacts**

- artifact ID/job ID/role/path/MIME/bytes/hash；
- remote expiry、本地 cache path/cache state；
- verified timestamp、lease/ref count、last access。

**job_events**

- monotonic sequence、timestamp、old/new status、stage、message、details；
- 用于 UI 恢复和 AgentScape 事件追踪，不记录 Secret/原图内容。

**metadata/migrations**

- DB schema version、app version、migration timestamp。

状态机：

```text
draft → uploading → queued → running → downloading → validating → succeeded
                         │          │             │
                         ├→ cancel_requested → cancelled
                         ├→ connection_required
                         ├→ failed
                         └→ expired
```

`connection_required` 是可恢复阻塞，不是远端终态；`succeeded` 只有在 result envelope 校验完成后出现。

### C3D-06：恢复、取消、重试与幂等

任务：

1. Agent 启动和 Modal 连接成功时自动 reconcile 非终态。
2. 对 generic Modal FunctionCall 与 EmbodiedGen workflow job 使用 provider adapter，不在 JobManager 写大量分支。
3. 指数退避带 jitter；前台 UI 轮询不能造成多个并发 poller。
4. 远端异常映射到 `auth/input/quota/queue/model/artifact/network/internal`。
5. cancel 拆为 `cancel_requested`、`cancel_acknowledged`、`cancelled`；无法中断的阶段继续观察终态。
6. retry 默认创建新 Job，并通过 `retry_of` 连接；只有明确可恢复 stage 才原 Job resume。
7. idempotency key 冲突必须拒绝，不能重复计费。
8. Agent 异常退出时 SQLite transaction 保证最后事件与状态一致。

### C3D-07：Artifact Cache 与本地项目工作区

建议本地目录按内容和项目分离：

```text
app-data/
  jobs.sqlite3
  cache/sha256/<prefix>/<digest>
  projects/<project-id>/project.json
  logs/
```

计划要求：

- 下载到临时文件，边流式写入边 hash，验证后原子 rename；
- cache 文件名不信任远端 artifact name；
- project 只保存引用、用户命名、参数和 lineage，不复制大文件；
- 用户“另存为”才复制到外部目录；
- cache eviction 尊重 active job、打开的 Viewer、AgentScape transfer lease；
- Volume 远端 TTL 与本地 cache TTL 分开；
- 清理前预估释放空间并允许保留已收藏结果；
- 不允许 `/v1/assets?path=` 读取任意合法相对路径，最终收敛到 job-scoped artifact ID。

### C3D-08：React 显式工作流状态机

当前 `App.tsx` 用多个 state 和长 while-polling 完成 MVP。计划拆为可恢复状态：

- connection；
- source validation；
- preprocessing；
- candidate selection；
- generation form；
- job observer；
- result/viewer；
- history/project。

要求：

1. 刷新/窗口重建后从 Agent 获取真实状态，不从 React memory 猜。
2. 离开页面不取消 Job。
3. 同一时间允许查看多个 Job，提交是否并发由云端 capability 决定。
4. UI 明确展示 queued/running/downloading/validating，不只显示“生成中”。
5. 错误同时提供用户说明、稳定 code、可执行下一步；trace 只在诊断面板显示。
6. 所有 object URL、Three 资源和 polling subscription 在卸载时释放。

### C3D-09：Viewer 与本地结果验证

现有 `GlbViewer.tsx` 已有 GLTFLoader、自动取景、OrbitControls 与 dispose。补齐：

- GLB 解析错误分类；
- 几何、材质、纹理、动画、bounds、文件大小摘要；
- non-finite transform/geometry 警告；
- 网格/纹理 budget 与 AgentScape budget 对照；
- ground/grid、单位与轴向提示；
- wireframe/材质检查模式和标准截图；
- Viewer 失败不删除 artifact；
- 下载按钮只下载 hash 已验证的本地 bytes；
- “发送到 AgentScape”走 Connector artifact handoff，而不是临时 blob URL。

### C3D-10：凭据与 loopback 安全

现有随机端口、256-bit session token、origin allowlist 和 Windows vault 是正确基础。继续加固：

1. Agent 强制 bind `127.0.0.1`，拒绝外部接口配置。
2. session token 只通过 Tauri invoke 交给主窗口，不写普通日志/DB。
3. CORS 不是认证替代；每个请求仍校验 session/pair token。
4. AgentScape 配对生成独立短期 token，不复用 Tauri 主 token。
5. 配对 token 绑定 origin、scope、过期时间，可立即撤销。
6. 下载 endpoint 绑定 artifact/job/owner，支持一次性或短 lease。
7. 响应错误不返回 Modal token、环境、绝对路径或远端 traceback。
8. Windows 凭据删除后验证 vault 与运行中 session 的语义：默认仅删除持久化副本，不隐式取消正在执行的 Job。
9. 日志滚动、大小上限、诊断导出脱敏。

### C3D-11：Local Modal Connector API

Connector 不直接暴露底层 `/modal/connect` 给 AgentScape。提供产品级资源：

| 资源 | 责任 |
|---|---|
| `/connector/v1/session` | 配对、scope、版本协商、撤销 |
| `/connector/v1/capabilities` | modal-3D 与 EmbodiedGen 统一能力 |
| `/connector/v1/jobs` | 提交 operation、幂等 key、查询列表 |
| `/connector/v1/jobs/{id}` | 状态、stage、progress、错误、result |
| `/connector/v1/jobs/{id}/cancel` | 明确取消请求 |
| `/connector/v1/artifacts/{id}` | job-scoped 流式 bytes、range/hash headers |
| `/connector/v1/events` | SSE 或等价事件通道；断线可用 sequence 恢复 |

首版只开放 AgentScape 所需 operation，不提供任意 Modal function/app/path 调用。Connector 必须能够在没有桌面主 UI 打开的情况下，由 Tauri 管理的 Agent 持续服务；是否允许独立后台启动由后续产品决策决定。

### C3D-12：可观测性与诊断

统一 correlation：`client_session_id → local_job_id → provider request_id → remote_job/call_id → artifact_id`。

指标与日志：

- sidecar start/handshake/connect；
- upload/download bytes、hash、duration；
- queue/run/validate wall time；
- poll/reconcile/cancel attempt；
- capability revision/cache age；
- DB migration/recovery；
- Viewer parse/resource summary；
- AgentScape pairing/access audit。

隐私约束：默认不记录 prompt 全文、concept 全文、图片内容、token、绝对用户路径。用户主动导出诊断包时仍要脱敏。

### C3D-13：打包、升级与兼容

任务：

1. 锁定并记录 Node/Rust/Python/Modal SDK/PyInstaller 组合。
2. sidecar 构建产物生成 hash 和版本 manifest。
3. Tauri 启动前检查 sidecar version 与 DB migration compatibility。
4. 安装升级不得清除 app-data、项目、Job 或 Windows credential。
5. sidecar 启动失败显示脱敏尾部日志，并可一键导出诊断。
6. Windows 10/11、非 ASCII 用户目录、无管理员权限、只读安装目录均 smoke。
7. Modal SDK 升级先在打包 sidecar canary，再进入正式安装包。
8. 非 Windows 开发模式明确“不支持安全持久化”或接入相应系统 keyring，不能退化为明文文件。

### C3D-14：测试计划

**Python 单元/组件测试**

- path、MIME、alpha、hash；
- capability compatibility/fallback；
- model/profile option；
- SQLite create/migrate/corrupt/recovery；
- Job 状态机、幂等、cancel、expired/local cache；
- provider error normalization；
- artifact streaming/range/hash/lease；
- connector session scope。

**Rust 测试**

- token entropy/handshake 文件权限；
- child crash/timeout/stop/tree kill；
- app-data 注入；
- credentials store/load/delete；
- upgrade keeps DB and cache；
- log cleanup。

**React 测试**

- capability-driven model/profile；
- RGB 与 direct RGBA；
- candidate selection；
- restored job/history；
- connection_required/reconnect；
- cancel_requested；
- Viewer error/resource cleanup；
- UI 不显示 Secret。

**契约测试**

- 与 `modal-3D` capability/job/result fixtures 双向兼容；
- 与 EmbodiedGen workflow fixtures 兼容；
- AgentScape connector consumer fixtures；
- 旧 client 遇到新 capability 的降级行为。

**打包 E2E**

- 干净 Windows VM 安装；
- 保存并恢复凭据；
- 自有 RGB → SAM → 四模型各一个 canary；
- direct RGBA → 跳过 SAM；
- 生成中杀死/重启应用并恢复；
- 成功后断网仍能打开本地 cache；
- AgentScape 配对 → submit → bytes → revoke。

## 6. 建议的实施顺序

1. C3D-00 基线与 smoke。
2. C3D-01/02 capability 和输入事实统一。
3. C3D-05/06 完成 DB、恢复、幂等和取消。
4. C3D-07 本地 artifact cache，消除远端结果过期风险。
5. C3D-03/04 完善 SAM 模式与模型 options。
6. C3D-08/09 UI 状态机、历史和 Viewer 加固。
7. C3D-10/11 配对式 Connector。
8. C3D-12/13/14 诊断、打包和完整 E2E。

Connector 不能早于 Durable Job 与 Artifact Cache 对外承诺稳定；否则 AgentScape 会依赖不可恢复的临时行为。

## 7. 第一组完成定义中的客户端部分

- 用户可以选择普通 RGB 或自备 canonical RGBA；
- Cloud SAM 可交互，direct RGBA 可跳过；
- 四模型和 profile 来自云端能力清单；
- Job 跨窗口、Agent 重启和 Modal 重新连接恢复；
- 结果在远端过期前进入 hash-verified 本地 cache；
- GLB 可预览、另存、重复使用，失败有明确诊断；
- Modal Secret 不进入 React、浏览器存储、日志或项目文件；
- Connector 以最小 scope 服务 AgentScape；
- 干净 Windows 安装包完成真实 E2E；
- 所有状态均来自本地/远端事实，不用 UI 文案伪造成功。

## 8. 非目标

- 基础安装包内置数十 GB 本地 SAM/3D 模型；
- 浏览器直接调用私有 Modal API；
- 第一阶段编辑网格、骨骼或 PBR 材质；
- 把远端 Volume path 当作可永久分享 URL；
- 用客户端参数覆盖云端未声明的实验 kwargs。
