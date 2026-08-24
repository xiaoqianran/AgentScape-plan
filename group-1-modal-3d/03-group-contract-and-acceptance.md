# 第一组联合契约与验收：`modal-3D + modal-3D-client`

> 执行进度、当前 HEAD、已验证证据与下一批具体任务见 [`00-execution-status.md`](./00-execution-status.md)。本文件保留为设计/验收基线。

## 1. 联合交付目标

第一组完成时，用户应能在一台全新 Windows 设备上完成下面的事实闭环：

```text
用户自己的 RGB / RGBA
→ 本地校验与可选 SAM
→ 选择任一可用 Modal 3D 模型
→ 可恢复异步任务
→ hash-verified GLB
→ 本地预览、保存、再次打开
```

任何一个环节只靠手工运行脚本、复制 Volume path 或打开 Modal 控制台，都不算产品闭环通过。

## 2. 两个仓库的责任分界

| 事实/行为 | `modal-3D` | `modal-3D-client` |
|---|---|---|
| 模型 ID、部署、输入、option schema | 唯一事实源 | 发现、缓存、渲染、兼容校验 |
| SAM 模型和候选语义 | 实现和云端事实源 | provider 选择、用户交互、恢复 |
| Modal FunctionCall | 创建和返回 remote ID | 保存、轮询、取消、恢复 |
| 原始云端 artifact | 生成、校验、Volume commit | 下载、hash 复验、本地 cache |
| Modal Secret | 不接收用户明文 API | Tauri vault → Python memory |
| 本地 Job ID/历史 | 不拥有 | 唯一用户可见身份 |
| UI/Viewer | 不拥有 | 产品责任 |
| AgentScape 配对 | 不拥有 | Local Connector 责任 |

## 3. 版本协商

每次连接必须协商四类版本：

- capability schema；
- generation request/result contract；
- SAM selection/materialization contract；
- local Connector contract。

兼容规则：

1. 主版本不一致即 incompatible，相关操作禁用。
2. 新增可选字段允许旧客户端忽略。
3. 删除/改义字段必须升主版本。
4. model/workflow 自身 revision 与 contract version 分开。
5. 客户端保存提交时的 capability hash；恢复时不能用新默认参数解释旧 Job。

## 4. Capability Contract

`modal-3d.capabilities.v1` 至少表达：

### 服务层

- service/contract version；
- deployment/environment/revision；
- generated/fetched timestamp；
- service status 与兼容范围。

### 模型层

- stable model ID、display name、status；
- input types、alpha requirement、bytes/pixel limits；
- output roles/MIME、texture/geometry 特征；
- profiles 与 option JSON Schema；
- seed 支持；
- historical performance class；
- upstream/build/release provenance；
- warnings/deprecation。

### SAM 层

- segment/refine/materialize 可用性；
- prompt/box/candidate limits；
- output sizes；
- scene/selection TTL；
- schema version。

验收：删除客户端本地某个模型硬编码后，云端 capability 新增/disable 模型可被 UI 正确反映，不需重新编译才能隐藏 disabled 模型。

## 5. Input Descriptor

所有输入统一为 descriptor，而非裸 path：

| 字段组 | 必需内容 |
|---|---|
| identity | artifact ID、SHA-256、bytes、MIME |
| image | width、height、channels、alpha summary |
| storage | 受控 namespace 内 relative path，或由 Connector 解析后的 opaque location |
| lineage | source kind、source hash、SAM scene/selection/candidate（如有） |
| policy | created time、TTL/retention class、owner request |

安全规则：

- path 只允许 `client-inputs/`、`modal-2d-primary/`、`sam31/` 等声明 namespace；`modal-2d-primary/` 必须由 2D primary manifest/可信 handoff 创建，不能接受任意同名前缀；
- absolute、反斜线逃逸、`..`、NUL、超长路径一律 CPU 层拒绝；
- MIME 由内容检测，不信任扩展名；
- Worker 必须验证 descriptor 与实际文件一致，至少核对 bytes/hash 或受信任 commit manifest；
- 用户本地文件名不进入远端真实路径。
- 2D preview role 一律不能冒充 downstream primary；不共享 Volume 时由 Connector 做 hash-verified bridge，不能把跨部署 path 当成天然可读。

## 6. SAM Contract

### Segment/Refine 结果

- `scene_id` 与 source image hash；
- `selection_id` 与 concept/boxes hash；
- image size；
- ordered candidates；
- candidate ID、score、normalized bbox、mask stats；
- model/schema revision；
- expires_at。

### Materialize 结果

- scene/selection/candidate IDs；
- canonical artifact descriptor；
- output size、foreground bounds、alpha stats；
- deterministic request hash；
- warnings。

恢复要求：客户端重启后，只要 TTL 未过，可以重新获取 selection 或 materialize；TTL 已过则提供“重新分割”，不能把 404 显示为未知失败。

## 7. Generation Request

逻辑请求字段：

- client request/local job/idempotency ID；
- model ID；
- input descriptor/path reference；
- profile ID；
- normalized/effective options；
- capability/contract version；
- optional retention/priority metadata。

校验顺序必须是：

1. contract/version；
2. model status；
3. path namespace；
4. input metadata/alpha；
5. profile/option；
6. idempotency；
7. 才 spawn GPU Worker。

## 8. Job Contract 与状态映射

### 云端 Gateway

云端只承诺：accepted request、remote call ID、model、deployment 和最终 result/exception。它不虚构细粒度模型进度。

### 本地 Connector

本地拥有完整用户状态：

| 本地状态 | 远端事实 | UI 语义 |
|---|---|---|
| `uploading` | 尚未提交 | 正在准备输入 |
| `queued` | 已接受、未观察到执行 | 已排队 |
| `running` | FunctionCall 未终结 | 云端生成中，不显示虚假百分比 |
| `connection_required` | 无法查询但没有失败证据 | 连接 Modal 后恢复 |
| `cancel_requested` | 已请求取消，终态未知 | 正在取消 |
| `downloading` | 远端成功、本地未缓存 | 正在取回结果 |
| `validating` | bytes 已有、hash/GLB 未验完 | 正在校验 |
| `succeeded` | result + 本地 artifact 验证通过 | 本地可用 |
| `failed` | 稳定失败证据 | 显示 code/stage/retry |
| `cancelled` | 取消终态 | 已取消 |
| `expired` | 远端与本地均无可用结果 | 结果过期 |

禁止用轮询超时直接把 Job 标为 failed。

## 9. Result/Artifact Contract

### Result Envelope

- request/job/call/model/deployment identity；
- input/options/seed lineage；
- primary artifact ID；
- artifacts 数组；
- timing 与 worker-specific metrics；
- geometry/texture summary；
- validation；
- warnings；
- created/expires timestamps。

### Artifact Descriptor

- opaque artifact ID；
- role，例如 `primary-glb`、`manifest-json`；
- safe display name；
- relative storage path（仅 Connector 内部可见）；
- MIME、bytes、SHA-256；
- created/expires；
- producer stage/model/revision。

客户端 success 门：

1. descriptor 完整；
2. 流式下载 bytes 数量一致；
3. SHA-256 一致；
4. GLB header/结构可解析且 scene 非空；
5. 原子写入本地 cache；
6. Job 与 artifact DB transaction 关联完成。

## 10. Error Contract

错误最少包含：

- stable `code`；
- category；
- stage；
- retryable；
- user message；
- correlation ID；
- optional safe details。

首批错误类别：

| 类别 | 示例 | 默认动作 |
|---|---|---|
| auth | token 无效/权限不足 | 重新连接，不自动重试 |
| input | 损坏图片、无 alpha、过大 | 用户修正 |
| capability | model disabled/schema incompatible | 刷新能力或升级 |
| queue | 调度/容量暂不可用 | 有限退避重试 |
| model | OOM、推理/导出失败 | 可复制参数重试或换模型 |
| artifact | commit、missing、hash mismatch | 重新下载；不成功则失败 |
| network | 本地/Modal 连接中断 | 保持可恢复状态 |
| cancelled | 用户取消 | 不提示为错误 |
| internal | 未分类 | 保留 correlation，脱敏诊断 |

远端 Python traceback 不能直接成为 UI message。

## 11. 模型验收矩阵

每个模型必须用同一 canonical RGBA canary 和模型专属边界用例：

| 模型 | 正向验收 | 专属负向/预算验收 |
|---|---|---|
| FastSAM3D++ | recommended profile 输出有效 GLB | alpha 无意义；DMD option 越界 |
| Hunyuan2.1++ | 50-step profile 输出有效 GLB | steps/interval/history 越界；无 alpha |
| Hermit-TRELLIS2++ | geometry GLB、seed lineage | 输出能力不夸大为 PBR；空 geometry |
| Pixal3D | textured/PBR GLB 可预览 | fov 越界；4K texture 和 100 MiB budget 提示 |

每个模型至少记录：冷启动 wall、load、inference、export、GLB bytes、vertices/faces、texture summary、GPU peak。性能基线只用于回归比较，不在没有样本分布前写硬 SLA。

## 12. 端到端验收场景

### E2E-01：普通照片成功

- JPEG 小于限制；
- Cloud SAM 找到候选；
- 用户选择候选并 materialize；
- 选择 FastSAM3D++；
- Job 成功、hash 匹配、Viewer 打开、另存 GLB；
- lineage 可追到 source hash 和 candidate。

### E2E-02：自备 RGBA 跳过 SAM

- alpha 有前景/背景；
- 本地校验通过；
- Hunyuan2.1++ 成功；
- history 明确 preprocess=`skip`。

### E2E-03：四模型可发现

- 不重编译 client；
- `/v1/models`/capability 显示四个模型及状态；
- 每个 recommended profile 可提交；
- disabled 模型不可提交。

### E2E-04：应用重启恢复

- Job running 时强制退出桌面应用；
- 重启后 DB 记录存在；
- 恢复凭据后 reconcile；
- 成功结果下载并展示；
- 不产生第二次远端提交。

### E2E-05：网络/认证中断

- 轮询中断开网络或撤销 token；
- Job 进入 connection_required，不是 failed；
- 重新认证后恢复；
- 若远端已完成，继续下载。

### E2E-06：取消竞态

- running 时取消；
- UI 先显示 cancel_requested；
- 若远端先成功，按实际结果进入 downloading/succeeded，并记录 cancel race；
- 若远端确认取消，进入 cancelled；
- 不伪称已取消仍在运行的阶段。

### E2E-07：远端输出过期

- 已成功且本地 cache 完整：断开远端后仍可打开；
- 未缓存且远端过期：进入 expired，提供重跑；
- 两种情况不可混淆。

### E2E-08：Artifact 损坏

- 下载中断可安全重试；
- hash mismatch 不原子提交到 cache；
- Job 不进入 succeeded；
- 原临时文件被安全清理。

### E2E-09：打包安装

- 全新 Windows 非管理员安装；
- sidecar 正确启动；
- 非 ASCII 用户目录；
- 凭据保存/恢复；
- 真实生成、预览和下载均通过。

### E2E-10：AgentScape 最小 Connector

- 用户在 Tauri 允许配对；
- AgentScape 只能读允许的 capabilities、提交允许 operation；
- 获取 artifact bytes 后可撤销 token；
- 撤销后新请求 401，既有云端 Job 仍由本地历史可管理。

## 13. 非功能门禁

### 安全

- React/localStorage/IndexedDB 中无 Modal token；
- Agent 只监听 loopback；
- session/token 全部有 scope 和过期；
- path traversal、跨 job artifact 读取、恶意 MIME 均有测试；
- 日志/诊断无 Secret 和原始输入内容。

### 可靠性

- 任务提交成功后 React 崩溃不影响远端任务；
- DB migration 失败时保留原 DB 备份并拒绝带损坏状态运行；
- 清理不会删除 active/leased/favorite artifact；
- 相同 idempotency key 不产生重复 call。

### 资源

- 上传/下载流式处理，不把 100 MiB GLB 多份复制进内存；
- Viewer 释放 WebGL/texture/object URL；
- 本地 cache 有上限与可解释清理；
- 云端保持每 Worker `max_containers=1` 的初始成本策略。

## 14. Gate 通过条件

第一组只有在以下证据全部存在时才可作为 AgentScape 的前置依赖：

1. capability、request、result、artifact fixture 已版本化；
2. RGB 与 direct RGBA 两条链均通过；
3. 四模型真实 canary 通过；
4. SQLite restart/reconcile 通过；
5. artifact 本地 cache/hash 通过；
6. Windows 安装包通过；
7. Connector pairing/least privilege 通过；
8. 所有失败场景返回稳定 error code；
9. 文档中的模型数量、部署名和实际一致；
10. 没有修改 `EmbodiedGen` 才能完成第一组。
