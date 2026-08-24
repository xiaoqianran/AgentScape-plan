# 2D 组联合契约与验收

## 1. 联合目标

```text
Prompt（可选本地AI优化并确认）
→ modal-2d-client durable Job
→ modal-2d Gateway
→ SANA 或 Z-Image Worker
→ lossless primary + preview + manifest
→ 本地Gallery/cache
→ 可选2D→3D downstream pipeline
```

## 2. 仓库责任

| 责任 | `kaggle-inference-hub` | `modal-build` | `modal-2d` | `modal-2d-client` |
|---|---|---|---|---|
| 行为/历史基线 | 迁移源，只读归档 | 否 | 否 | legacy import |
| native/runtime构建 | 旧Notebook证据 | 唯一正式构建 | 消费release | 否 |
| 模型/option capability | 旧硬编码 | compatibility metadata | 唯一云端事实 | 发现/缓存/表单 |
| 推理 | 迁移期Kaggle | 否 | 正式Modal Worker | 提交/观察 |
| Prompt AI | 旧Hub | 否 | 否 | Local Connector |
| 用户Job/history | 旧SQLite | 否 | remote call/result | 本地唯一身份 |
| Artifact | 旧outputs | release artifacts | private Volume | local cache/project |
| 2D→3D | 旧URL→TripoSR | 否 | 产出primary descriptor | pipeline编排/handoff |

## 3. 契约版本

- `modal-2d.capabilities.v1`；
- `modal-2d.generation.v1`；
- `modal.artifact.v1`（跨2D/3D中性）；
- `modal.local-job.v1`；
- `modal.pipeline.v1`；
- `modal.connector.v1`。

2D/3D客户端统一前，两边必须用同一fixtures验证后三项。

## 4. Text→Image Request

字段组：

- identity：request/idempotency/model/contract；
- text：confirmed prompt、protected source provenance reference；
- image：width/height/profile；
- sampling：seed/steps/guidance/negative等schema允许项；
- retention；
- capability hash。

服务端记录effective options。Prompt AI provider/key不进入云端request；可以记录已确认prompt来自哪个本地transform的非敏感摘要。

## 5. Image Artifact

### Primary

- role=`primary-image`；
- lossless PNG或明确lossless格式；
- width/height/color mode；
- SHA-256/bytes/MIME；
- 适合downstream=`image-to-3d`；
- model/revision/seed/options lineage。

### Preview

- role=`preview-image`；
- 可有损；
- 原primary hash引用；
- 只供UI；
- 明确禁止默认作为3D input。

### Manifest

- request/result identity；
- artifacts；
- timings/metrics；
- validation/warnings；
- upstream/release/weights；
- expires/retention。

## 6. Batch Contract

Batch是本地组合，不是单个GPU请求：

- parent ID/request；
- ordered child IDs；
- per-child prompt/seed/idempotency；
- max active submissions；
- counts/status；
- batch cancel policy；
- retry failed children；
- provenance mapping保持输入顺序。

批量Prompt AI逐项结果与GPU child一一对应，换行折叠不能偷偷增加任务。

## 7. 2D→3D Pipeline Contract

Parent pipeline：

- source 2D Job/artifact hash；
- preprocess policy=`sam-cloud/local/skip`；
- target 3D provider/model/workflow/profile；
- child Job IDs；
- intermediate canonical RGBA artifact；
- final 3D artifact/bundle；
- status/error/retry boundary；
- full lineage。

阶段：

```text
source-ready
→ preprocess-required / awaiting-candidate / canonical-ready
→ 3d-queued / 3d-running
→ downloading / validating
→ succeeded | provisional | failed | cancelled
```

用户交互候选选择是合法暂停状态，不是Job失败。

## 8. Error 语义

区分：

- Prompt AI失败；
- capability/schema；
- input/prompt/size；
- Modal auth/network；
- model load/inference；
- image encode/commit/hash；
- local download/cache；
- downstream SAM/3D；
- legacy import。

AI优化失败不自动计费提交；preview失败但primary有效可warning；primary失败则2D Job不能succeeded。

## 9. 验收场景

### 2D-E2E-01 SANA

- Prompt AI关闭；
- fixed prompt/seed/recommended；
- Job恢复；
- lossless primary/preview/hash；
- Gallery离线重开。

### 2D-E2E-02 Z-Image

- exact release/weights；
- 8-step profile；
- `sd-server` crash负向；
- primary/preview/result一致。

### 2D-E2E-03 Prompt AI

- saved key从vault恢复到Agent；
- source→processed→手改；
- 用户确认；
- cloud Job只看到最终确认prompt；
- key不在React/DB/log。

### 2D-E2E-04 Batch restart

- 10项、限流提交；
- 中途退出；
- 重启恢复children；
- 失败2项单独重试；
- 成功项不重复。

### 2D-E2E-05 Kaggle import

- snapshot旧SQLite/outputs；
- duplicate/missing/corrupt；
- import report；
- legacy Gallery离线可看；
- 原目录mtime/content不变。

### 2D-E2E-06 2D→3D

- SANA primary而非preview；
- Cloud SAM candidate；
- modal-3D选定模型；
- final GLB hash/Viewer；
- lineage贯穿source prompt、2D seed、candidate、3D seed/model。

### 2D-E2E-07 取消竞态

- 2D running取消；
- cancel_requested与实际终态；
- parent batch/2D→3D只传播到未终结child；
- 已成功artifact保留。

### 2D-E2E-08 安全

- 无默认token；
- 无localStorage Secret；
- loopback session；
- path/artifact scope；
- prompt/error安全渲染；
- hash mismatch/oversize拒绝。

## 10. Gate 通过条件

1. `modal-build` 两模型供应链固定；
2. `modal-2d` 两Worker/Gateway canary；
3. `modal-2d-client` single/batch/restart/cache；
4. Prompt AI安全迁移；
5. lossless primary contract；
6. Kaggle shadow质量/性能报告；
7. legacy history import；
8. 2D→3D跨客户端契约通过；
9. 共享Job/Artifact fixtures与3D客户端一致；
10. Kaggle生产Tunnel可在回退窗口后撤销。
