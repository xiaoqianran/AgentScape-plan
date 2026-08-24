# `modal-2d-client` 新仓实施计划

## 1. 项目定位

`modal-2d-client` 首先提供独立可验收的 2D 产品工作区，最终成为统一 2D/3D 桌面产品中的 `2D Studio` 模块。它不是长期复制一套 `modal-3D-client` 基础设施。

首版职责：

- Tauri/本地 Connector 安全使用 Modal credential；
- capability驱动的 SANA/Z-Image Prompt→Image；
- single/batch durable Job；
- Prompt AI、Gallery、历史、下载、项目 artifact；
- 将 lossless 2D artifact 交给 3D pipeline；
- 为后续 AgentScape Connector提供统一 Job/Artifact投影。

## 2. 从 Kaggle Hub 迁移的产品能力

### 保留并重做

- Prompt Studio：single/batch；
- enhance/creative/translate/clean；
- target-model prompt adapter；
- source/processed/edited provenance；
- model topbar/capability；
- queue/running/result/failed status；
- Gallery、image preview、下载；
- result“转 3D”；
- worker/服务诊断改成 Modal deployment/Connector诊断。

### 删除/替换

| 旧实现 | 新实现 |
|---|---|
| `kaggle_hub_token` localStorage | OS credential vault → Python memory |
| `/task`/worker claim | Connector submit → Modal Gateway FunctionCall |
| WebSocket broadcast only | durable Job DB + SSE/poll recovery |
| `/images` public static URL | job-scoped artifact bytes + local cache |
| WebP quality 90唯一结果 | lossless primary + preview |
| Hub SQLite queue | Connector SQLite Job/Event/Artifact store |
| Cloudflare Tunnel | private Modal RPC |
| Worker control center | capability/deployment/cost/job diagnostics |

## 3. 建议技术栈

- Tauri 2；
- React + TypeScript；
- Python 3.12 FastAPI Local Connector；
- SQLite durable state；
- OS keyring/Windows Credential Manager；
- TanStack Query用于服务端投影，React Hook Form + schema-driven options；
- 本地 artifact cache按 SHA-256；
- 与 `modal-3D-client` 相同的 loopback session、错误、Job、Artifact contract。

版本可与现有前端逐步收敛；不能仅为复用 Kaggle UI 强制把统一客户端同时升级全部依赖。

## 4. 过渡期结构

```text
modal-2d-client/
  src/
    workspaces/2d/        Prompt、Batch、Gallery、Image Detail
    shared/               typed connector client/UI primitives
  agent/
    providers/modal_2d.py
    prompt_pipeline/
    jobs/artifacts/projects（使用统一 schema）
  src-tauri/
    thin host（过渡期）
  migrations/
    kaggle_legacy.py      只读导入逻辑
```

共享基础的最终位置由客户端统一计划确定。首版可以有独立 shell，但所有持久 schema、endpoint、provider interface 必须中性命名，不使用 `KAGGLE_*` 或 `3D-only` 字段。

## 5. 用户工作流

### 5.1 单 Prompt

1. 连接 Modal；读取 capability。
2. 选择 SANA/Z-Image、profile、尺寸、seed。
3. 可选本地 Prompt AI；展示 source/processed，用户确认。
4. 创建 local Job，提交 Modal。
5. 显示 queued/running/downloading/validating。
6. 下载 lossless primary和preview，验证 hash并缓存。
7. Gallery可预览、下载、复制参数、衍生为3D。

### 5.2 Batch

1. 一行一个 prompt或结构化导入；空行过滤。
2. AI优化逐项结果，失败项回退原文但显式标记。
3. Batch是本地 parent Job；每张图是 child Job、独立seed/idempotency。
4. 可限制提交并发/每日预算；不一次创建不可控数量远端调用。
5. 单项失败可重试，成功项不重复生成。
6. Batch完成状态包含 succeeded/failed/cancelled计数，不用全有或全无。

### 5.3 2D→3D

1. 用户在 Gallery 选择 lossless primary。
2. 选择 3D策略：Cloud SAM后模型、直接RGBA、或将图交 EmbodiedGen Image→3D。
3. 创建 pipeline parent Job；2D artifact作为 immutable input。
4. 切换到3D工作区继续候选选择/模型配置，或选择已批准auto策略。
5. 3D结果保留 `derived_from` 2D Job/artifact/prompt/seed。

在客户端尚未统一前，handoff通过共享 Local Connector/neutral artifact ID完成，不使用公网URL或剪贴板路径。

### 5.4 历史 Kaggle导入

1. 用户选择旧Hub目录/SQLite；
2. 只读扫描和预览报告；
3. 计算hash并导入cache/history；
4. 标记provider=`kaggle-legacy`；
5. 缺文件/损坏不阻塞其他项；
6. 不把旧queued/inflight任务恢复为Modal任务；
7. 不读取/导入旧token。

## 6. 工作包

### C2D-00：共享 Contract 先行

在创建大量UI前，与 `modal-3D-client` 对齐：

- local Job ID/state/error；
- provider/operation/remote identity；
- artifact descriptor/cache/lease；
- capability cache/version；
- project/lineage；
- loopback session；
- Connector endpoint。

同一 fixture必须被两客户端消费。若字段只适用于3D或2D，放 typed operation payload，不污染公共Job表。

### C2D-01：Tauri/Connector安全基础

- 随机loopback port/session token；
- sidecar lifecycle/handshake/log；
- Windows credential vault；
- 已保存credential直接注入Agent，不回传React；
- app-data/jobs/cache/projects；
- no localStorage Secret；
- 后续可无损迁移到单一统一Host。

不得从 Kaggle前端复制 `kaggle_hub_token` 状态。

### C2D-02：Modal 2D Provider

Python provider实现：

- fetch/cache capabilities；
- submit model/profile/options/idempotency；
- poll/cancel FunctionCall；
- normalize result/error；
- download primary/preview；
- reconnect/reconcile；
- model disabled/incompatible handling。

React不拼装模型特定kwargs。

### C2D-03：Durable Job/Batch

使用统一DB：

- parent/child relation；
- request hash/idempotency/retry_of；
- status/stage/timestamps；
- remote call；
- prompt lineage reference；
- result/artifact；
- cancel/recovery；
- event sequence。

Batch取消只取消未终结child；成功child保留。应用重启后从DB和Modal事实恢复。

### C2D-04：Artifact Cache/Project

- lossless primary与preview角色分开；
- 流式下载、hash、原子rename；
- image decode/dimensions/color validation；
- project引用不复制bytes；
- favorite/lease/eviction；
- 2D→3D handoff lease；
- historical Kaggle WebP 标有损；
- 用户另存支持PNG/WebP/JPEG选择，但不改变primary事实。

### C2D-05：Prompt Pipeline

- 从现有 `hub/prompt_pipeline`迁移行为fixture；
- settings/API key进OS vault；
- adapter从capability/model revision读取；
- 4种模式与语言策略；
- single/batch并发限制；
- source/processed/provider model/elapsed/edited/stale adapter；
- AI失败不自动提交原文，UI要求用户确认批量回退策略；
- 用户确认后才生成GPU Job。

### C2D-06：Capability驱动表单

字段：model、profile、width/height/aspect、steps、seed、negative prompt（若支持）、advanced options。

要求：

- 模型切换重置/迁移参数有明确规则；
- invalid组合在本地与云端双重校验；
- historical Kaggle默认值只在对应profile说明；
- 显示estimated duration/cost class而非假进度；
- disabled/degraded模型不可提交；
- seed series用于batch且记录每项effective seed。

### C2D-07：Gallery与Image Detail

每张卡片显示：model/revision、time、seed、size、steps/profile、job/admission、primary/preview、本地cache。

详情：

- 原图像素级查看与fit/1:1；
- prompt/source/processed provenance；
- result manifest/hash/bytes/color；
- copy prompt/options/seed；
- rerun/derive/download/favorite/delete local reference；
- “转3D”总是使用primary；
- 失败/取消Job在Job Center，不伪装Gallery result。

### C2D-08：Job Center与诊断

- single/batch树；
- stage/connection/cancel/retry；
- model capability/deployment health；
- local disk/cache；
- correlation ID；
- Kaggle legacy import状态；
- 日志导出脱敏。

不展示底层worker claim，因为Modal调度不是用户可操作队列。

### C2D-09：Kaggle Legacy Import

- schema/version探测；
- SQLite一致性只读打开；
- URL→文件path严格限制在选定root；
- hash/MIME/dimensions；
- duplicate by hash；
- prompt meta迁移；
- import transaction/report/resume；
- 原数据零写入；
- malicious path/payload/巨型JSON测试。

### C2D-10：2D→3D Handoff

在统一客户端前先实现中性API：

- `create_pipeline_from_artifact`；
- source artifact lease；
- target operation/capability；
- parent/child Job；
- UI deep-link只传opaque project/pipeline ID；
- 3D侧通过Connector查descriptor/bytes/path授权；
- pipeline完成/取消后释放lease；
- 重启恢复到正确工作区与stage。

### C2D-11：测试

**Python**：capability、provider、Job/batch、prompt、artifact、legacy import、handoff。

**Rust**：sidecar/vault/app-data/upgrade，与3D客户端使用同一contract suite。

**React**：Prompt AI确认、single/batch、model切换、history、Gallery、Job recovery、derive 3D、无Secret。

**E2E**：干净Windows→SANA/Z-Image→restart→Gallery→2D primary→3D Job；Kaggle legacy导入。

### C2D-12：打包与升级

- 独立产品MVP可打包，但清晰标注将合并；
- app-data/schema使用中性版本；
- unification installer可发现并迁移2D app-data；
- 卸载不默认删除artifact/history；
- sidecar/DB兼容性检查；
- 非ASCII路径/非管理员/Windows 10/11。

## 7. 实施顺序

1. C2D-00～04：共享契约、Provider、Job、Artifact。
2. C2D-05～08：Prompt Studio、表单、Gallery、Job Center。
3. C2D-09：legacy import。
4. C2D-10：2D→3D handoff。
5. C2D-11～12：E2E、打包、升级。

## 8. 完成定义

- SANA/Z-Image capability驱动生成；
- single/batch跨重启恢复；
- lossless primary与preview分离并hash验证；
- Prompt AI Secret不进React且用户确认后才提交；
- Gallery/历史/复制参数/下载/derive完整；
- Kaggle历史可只读导入；
- 2D primary可通过中性Artifact/Job直接进入3D；
- 公共contract与3D客户端一致；
- 没有Kaggle Tunnel、worker token、localStorage Secret；
- 独立shell能被后续统一客户端无损迁移。

## 9. 非目标

- 长期维护第二套凭据/Job/Artifact实现；
- 首版图片编辑器、ControlNet、LoRA、inpainting、video；
- Browser直接访问Modal；
- 自动把所有2D结果生成3D；
- 删除旧Kaggle历史。
