# `modal-3D` 文件级实施计划

## 1. 项目定位

`modal-3D` 继续保持“小而独立的 Modal 3D 部署层”：

- SAM 3.1 提供共享对象提取与 canonical RGBA；
- 每个 3D 模型拥有独立 App/Image/weight Volume；
- CPU Gateway 只负责校验、路由与异步提交；
- 所有结果写入统一 artifact Volume；
- 不承担桌面 UI、用户凭据、本地 Job DB 或 AgentScape 世界逻辑。

## 2. CodeGraph 现状

### 2.1 关键文件

| 文件 | 当前责任 | 当前问题 |
|---|---|---|
| `modal_3d/common.py` | `ModelName`、`generation_result()` | 模型列表与结果字段很小，但没有能力版本、option schema、artifact hash |
| `modal_3d/config.py` | `ModelSpec`、`MODELS` | 当前无调用者，与 `common.py`/Gateway 有重复事实 |
| `modal_3d/gateway.py` | `model + input_path + options → FunctionCall` | 只校验 enum；没有 capabilities、request ID、option 校验、输入 namespace 约束 |
| `modal_3d/sam3_1.py` | segment/refine/materialize | 已有扎实输入校验和持久化；未进入统一 pipeline/job contract |
| `fastsam3d_plus_plus.py` | RGBA→GLB | 要求有效 alpha；DMD 参数可配；结果有顶点/面/VRAM |
| `hunyuan2_1_plus_plus.py` | RGBA→GLB | 要求 pre-matted alpha；interval/history/steps 有部分校验 |
| `hermit_trellis2_plus_plus.py` | RGBA→GLB | 返回结构较少，内部 `preprocess_image=True` |
| `pixal3d.py` | RGBA→PBR GLB | `fov`/seed；4K texture；输出可能触发 Web budget |
| `workers/template.py` | Worker 模板 | 需要与真实统一契约对齐或明确仅供 scaffold |

### 2.2 当前稳定约束

- 4 个活动生成模型：`fastsam3d-plus-plus`、`hunyuan2.1-plus-plus`、`hermit-trellis2-plus-plus`、`pixal3d`。
- 一个共享预处理服务：`modal-3d-sam31`。
- Worker 输入通过 `/artifacts` 相对路径传递，拒绝 absolute/`..`。
- GPU 类型当前固定 L40S，GPU Worker `max_containers=1`。
- 权重在 CPU 函数同步，GPU 启动使用缓存，重型 CUDA wheel 来自 `modal-build` Release。
- GLB 结果统一写入 `modal-3d-artifacts`。

### 2.3 文档/事实不一致

README 的“Current workers”列出 4 个，但 Status 写“五个 L40S image-to-3D workers”。计划中的第一项审计要以部署与 `ModelName` 为事实，修正文档或补齐被遗漏的活动 Worker，不能让客户端靠 README 猜。

## 3. 目标模块边界

```text
capabilities.py       唯一模型/预处理能力定义
contracts.py          请求、结果、artifact、错误契约的纯函数校验
gateway.py            CPU submit/status metadata；不做模型推理
pipeline.py           可选 SAM→materialize→generation 的云端编排（第二阶段）
workers/*             模型 Worker；只消费 canonical input，返回统一原始结果
operations.py         canary、cleanup、artifact integrity（运维入口）
```

文件名为计划建议，最终实现可保持现有扁平结构；关键是责任不能重新混在一个模块里。

## 4. 工作包

### M3D-00：部署与事实审计

**目标**：建立“代码、文档、Modal 部署”三者一致的基线。

任务：

1. 记录每个 App 名、Function/Class 名、Volume 名、GPU、Python/CUDA/Torch、上游 commit、release tag。
2. 验证 4 个模型和 SAM 的实际部署名与 `WORKERS` 一致。
3. 列出每个 Worker 的必需 Secret；GPU Worker 不获得仅下载权重所需的 Secret。
4. 清理或标记 archive Worker，避免出现在 capabilities。
5. 修正“4/5 个 Worker”文档冲突。
6. 记录一组固定 canary 输入：普通 RGB、pre-matted RGBA、透明空图、超大图、损坏图。

验收：

- 一张 deployment matrix 可以唯一定位所有生产函数；
- `modal.Function.from_name()` 的名称均有自动 smoke；
- archive 不可能被 Gateway 路由。

### M3D-01：单一能力清单

**涉及**：`common.py`、`config.py`、`gateway.py`、所有 Worker。

建立 `modal-3d.capabilities.v1`，至少包含：

- `service_version`、`contract_version`；
- `models[]`：稳定 model ID、显示名、App/function、状态；
- 输入：`image/png|jpeg|webp`、是否必须 meaningful alpha、最大 bytes/pixels；
- 输出：GLB、是否带 PBR texture、预期 artifact 类别；
- option JSON Schema、default、min/max/enum；
- GPU/队列策略、预期 cold/warm 级别（只给分档，不把旧 benchmark 当 SLA）；
- upstream/release revision；
- preprocess 能力：SAM segment/refine/materialize；
- deprecation/compatibility 信息。

模型 profile 初始建议：

| Model | 输入前置条件 | 暴露参数 | 首版约束 |
|---|---|---|---|
| FastSAM3D++ | meaningful RGBA alpha | `seed`, `dmd_interval`, `dmd_history` | 校验 interval/history；默认关闭未证明收益的加速 |
| Hunyuan2.1++ | pre-matted RGBA | `seed`, `interval`, `history`, `num_inference_steps` | 为 steps 设置上下界；校验 alpha |
| Hermit-TRELLIS2++ | canonical RGBA 推荐 | `seed` | 明确当前 geometry/texture 能力，不暗示未输出 PBR |
| Pixal3D | canonical RGBA 推荐/要求 | `seed`, `fov` | fov 有物理范围；输出做 Web budget canary |

验收：

- Gateway、client Python、TypeScript UI 均从同一份 capability 生成/消费，不再手写 4 次模型 enum；
- `config.py` 不再是孤立死配置；
- capability schema 有 snapshot test 和向后兼容策略。

### M3D-02：输入契约与内容寻址

**目标**：让 Worker 永远只看到已验证、不可逃逸、可追溯的输入。

任务：

1. Artifact path 限制为受控 namespace，例如 `client-inputs/<sha256>.<ext>`、`modal-2d-primary/<sha256>.<ext>`、`sam31/...`、`jobs/...`；其中 2D namespace 只能由受信任 handoff/manifest 创建，不能由调用者任意拼 path。
2. Gateway 对 `input_path` 做 POSIX normalize、prefix allowlist、长度与字符校验。
3. 在提交前读取轻量 metadata：bytes、SHA-256、MIME/格式、像素、alpha extrema。
4. meaningful alpha 模型在占用 GPU 前失败，而不是在模型内部深处失败。
5. 输入 descriptor 写入 Job/结果 lineage，不只保留 path。
6. 同一内容重复上传保持幂等，不覆盖不同内容。
7. 定义最大输入 bytes/pixels，并与 client 一致。
8. 接收 `modal-2d` lossless primary 时核对 artifact role、producer、hash、MIME 与 retention lease；preview role 在 CPU 层拒绝。若两个部署不共享 Volume，由 Connector 按 hash 桥接到 `client-inputs/`，Worker contract 不感知传输差异。

验收用例：

- `/absolute`、`../escape`、空路径、错误 prefix 均在 CPU 层拒绝；
- 扩展名伪装、损坏图片、透明度全 255、透明度全 0 有确定错误码；
- 相同内容得到相同 input hash。
- 2D primary 可进入 SAM/3D，2D preview 即使扩展名与尺寸合法也不能作为默认 handoff 输入。

### M3D-03：统一 option 校验

当前 Gateway 将任意 `options` 传入 Worker，错误可能变成远端 Python `TypeError`。计划在 CPU 层按 capability schema 完成：

- 未知字段拒绝；
- bool 与 int 严格区分；
- seed 范围统一；
- 模型特定 min/max；
- 规范化后的 effective options 写入结果；
- option schema/version 写入 lineage。

不能采用“前端限制即可”的方式；AgentScape、CLI 和未来 API 都可能绕过 UI。

### M3D-04：统一 Job Envelope

`gateway.submit()` 继续异步 `spawn`，但返回内容扩展为稳定 envelope：

```text
request_id
call_id
model
contract_version
submitted_at
input descriptor
effective options
deployment revision
```

要求：

- 支持由 Connector 提供的 `request_id/idempotency_key`；
- 相同 key + 相同 payload 返回同一提交；
- 相同 key + 不同 payload 返回冲突；
- Gateway 不声称知道 FunctionCall 的实时进度，只给 `queued/running` 粗状态或由客户端轮询；
- 原有 `call_id + model` 在兼容窗口内保留。

### M3D-05：统一 Result 与 Artifact Manifest

扩展 `generation_result()`，但不把每个模型的 metrics 抹平。

必需字段：

- contract/model/deployment/input/options/seed lineage；
- primary artifact：path、name、MIME、bytes、SHA-256；
- timing：load、inference、postprocess、total（能测多少写多少，不伪造）；
- geometry summary：vertices/faces/bounds（能得出时）；
- texture summary：count/max dimension/estimated bytes（能得出时）；
- worker-specific metrics 独立保留；
- warnings 数组；
- `validation`：GLB parse、non-empty scene、finite geometry。

每个 GLB 旁写一份小型 JSON manifest。Artifact hash 必须在 commit 前计算，结果只有在 Volume commit 成功后返回。

### M3D-06：模型 Worker 收敛

逐 Worker 执行，不做大爆炸重构：

1. 保留各自独立 Image/Volume/`@modal.enter()`。
2. 把共享 path/input/result 逻辑收敛到纯 Python helper。
3. 每个 Worker 明确分段计时：load、preprocess、inference、export。
4. 输出前执行有限、便宜的 GLB structural check。
5. 捕获并转成稳定错误类别，同时保留内部 trace 供日志，不把敏感环境信息返回用户。
6. 模型 load 继续 offline；任何运行时远程下载视为部署失败。

逐模型专属事项：

- **FastSAM3D++**：锁定 `dmd_interval<=1` 与 DMD 开启两条路径；校验 `dmd_history`；记录 vertex-color/texture 模式。
- **Hunyuan2.1++**：在 CPU adapter 提前验证 alpha；限制 steps；确认 `mesh.export()` bytes 类型稳定。
- **Hermit-TRELLIS2++**：补全 model/upstream/gpu/seed/geometry metrics；明确导出当前是否无纹理。
- **Pixal3D**：固定工作目录清理；校验 fov；记录 texture size/PBR；对超 AgentScape budget 的 GLB给 warning 而不是静默。

### M3D-07：SAM 3.1 产品化接口

保留当前 bit-packed mask 设计，计划补充：

- capability 中公开 prompt 类型、候选上限、image limits；
- `scene_id/selection_id` TTL 与 cleanup；
- result JSON 加 schema/version/hash；
- refine box schema 明确中心坐标、normalized range、positive/negative；
- materialize 输出 descriptor 含 hash/MIME/alpha/bounds；
- 同一 scene/candidate/output size 的 materialization 幂等；
- scene cache hit 只作为 metric，不成为功能正确性依赖；
- client 断线后仍能凭 ID 恢复 selection。

第二阶段再决定是否在云端增加 `preprocess_and_submit` pipeline；第一阶段保留 client 显式选择 candidate 的交互优势。

### M3D-08：可观测性与成本控制

统一日志字段：

- request/call/job ID；
- model/deployment revision；
- input hash（不记录原图）；
- container instance/cold-warm；
- queue/start/load/infer/export/commit；
- artifact bytes/hash；
- GPU peak；
- normalized error code/stage。

资源政策保持：

- GPU `min_containers=0`、`max_containers=1`；
- 不启用 container input concurrency；
- CPU sync 也 `max_containers=1`；
- warm window 只有 benchmark 与真实流量证据才能改；
- 文档区分 cold scheduler wall、load、first/JIT、warm inference、postprocess。

### M3D-09：测试体系

当前 CodeGraph 没有发现生产代码覆盖测试，计划补齐：

**纯单元测试**

- capability schema；
- ModelName/route 唯一性；
- safe relative path/prefix；
- option normalize；
- result normalize/hash；
- alpha/input validation；
- error mapping。

**Mock Modal 合约测试**

- Gateway 正确定位 Function 并异步提交；
- 幂等 key；
- FunctionCall 异常；
- Volume reload/commit 失败。

**部署 canary**

- 每个 Worker：固定 RGBA → GLB；
- SAM：segment→materialize；
- Gateway：submit→FunctionCall→artifact；
- 下载后独立 GLB parser 校验；
- cold/warm 各一次但性能只作基线，不设未经验证的硬 SLA。

**负向 canary**

- 无 alpha；
- 错误 model/option；
- 过大图片；
- 缺权重；
- artifact commit 失败。

### M3D-10：部署与回滚

部署顺序固定：

1. schema/helper/test；
2. CPU weight sync；
3. 单 Worker 新 revision；
4. Worker direct canary；
5. Gateway capability 更新；
6. Gateway submit canary；
7. client capability compatibility；
8. 宣布启用。

回滚必须能只回滚一个 Worker，不影响其他模型；Gateway capabilities 可临时把模型标为 `disabled/degraded`，客户端不得继续展示为可提交。

## 5. 完成定义

`modal-3D` 第一阶段完成必须同时满足：

- 4 个活动模型和 SAM 能力可机器发现；
- 所有模型 option 在 CPU 层校验；
- 输入、Job、结果、artifact 都有 schema/version/hash/lineage；
- Client 不再硬编码模型事实；
- 每个 Worker 有正向与负向 canary；
- 一次 Worker 回滚不会破坏其余模型；
- 没有 GPU 运行时下载/编译；
- 文档与真实部署数量一致。

## 6. 后置事项

- 多 GPU 并发/提高 `max_containers`；
- 公网匿名 API；
- 模型自动质量排序；
- 多视图输入；
- 直接生成 AgentScape Manifest；
- 通用 3D 文件格式转换。

这些事项必须等待第一组契约稳定或由 AgentScape/EmbodiedGen 的真实需求触发。
