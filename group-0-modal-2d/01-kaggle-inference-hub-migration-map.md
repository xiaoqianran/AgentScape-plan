# `kaggle-inference-hub` 迁移地图

## 1. 迁移目标

`kaggle-inference-hub` 当前把本地 Hub、Kaggle 双 T4 Worker、2D/3D 模型、Prompt Pipeline、Gallery 和 Worker 协议放在同一仓库。迁移后的目标不是把 Notebook 改成 Modal Notebook，而是拆成与 3D 栈一致的三层：

```text
modal-build       固定/构建 2D runtime 与 release 产物
modal-2d          独立 Modal 2D Worker、Gateway、Capability、Artifact
modal-2d-client   本地 Tauri 产品、Prompt、Job、Gallery、结果与 Connector
```

`kaggle-inference-hub` 在迁移期只作为行为基线、Kaggle fallback 和历史数据源；完成后进入只读归档，不继续承担正式产品控制面。

## 2. CodeGraph 基线

2026-08-23 单独索引结果：62 文件、3,440 节点、14,834 边。

### 2.1 当前控制面

| 文件 | 当前责任 | 可继承语义 | 不应照搬 |
|---|---|---|---|
| `hub/app.py` | FastAPI、task/worker/upload/history/WebSocket/静态前端 | typed request、模型隔离、结果事件 | 巨型路由文件、Tunnel upload、共享 token、公开静态 artifact path |
| `hub/state.py` | SQLite WAL、原子 claim、lease、retry、worker/history | durable state、attempt、recovery、event ID | Kaggle worker claim 模型；Modal 已有调度/FunctionCall |
| `hub/config.py` | 3 个模型硬编码、限制、默认 token | stable model ID 与 input/output kind | 默认 `wangran`、模型事实重复、2D/3D混表 |
| `hub/crypto.py` | token SHA-256→AES-GCM 结果解密 | 传输完整性意识 | 用同一 Bearer Token 同时做认证和静态对称密钥 |
| `hub/prompt_pipeline/*` | OpenAI-compatible Prompt 编译 | 模式、模型 adapter、用户确认、provenance | API key 位于通用 Hub 进程或传到浏览器 |
| `frontend/*` | React Prompt Studio、batch、status、Gallery、3D preview | 产品交互和组件语义 | token 存 localStorage、同步 queue API、2D/3D强耦合类型 |
| `scripts/self_test.py` | SQLite、claim、HTTP、AES-GCM、TripoSR同步 smoke | 迁移 fixture/行为证据 | 只覆盖 TripoSR 为主的测试偏差 |

### 2.2 当前模型与归属

| 当前内容 | 迁移归属 | 决策 |
|---|---|---|
| `001-sana-sprint-1-6b.ipynb` | `modal-build + modal-2d` | 迁移；Diffusers resident Worker |
| `002-z-image-turbo-gguf.ipynb` | `modal-build + modal-2d` | 迁移；stable-diffusion.cpp release-only Worker |
| `003-triposr-image-to-3d.ipynb` / worker | `modal-3D` 候选 | 不进入 2D；先做与现有四模型价值/质量对比 |
| `004-kaggle-meshcoder.ipynb` | `modal-3D/modal-build` 研究候选 | 不因 Notebook 存在就发布 capability |
| `005-pixal3d-minimal-t4.ipynb` | `modal-3D` 历史证据 | 当前 Pixal3D Modal Worker 为生产事实源 |
| `006-trellis-2-*` | `modal-build/modal-3D` 研究/构建证据 | 与 Hermit-TRELLIS2++ release 对账后归档 |
| `007-fast-sam3d` / SAM3D body notebooks | `modal-3D`/EmbodiedGen研究 | 不进入 2D |
| Prompt Pipeline | 统一本地 Connector/client core | 2D 首用，后续可服务 3D prompt/workflow |
| React Prompt/Gallery UI | `modal-2d-client` | 迁移交互，不直接复制认证与 API |
| SQLite 历史/outputs | legacy import | 导入为只读历史 artifact，不恢复到新 Modal Job |

## 3. 当前 2D Worker 事实

### 3.1 SANA Sprint 1.6B

Notebook 当前：

- `Efficient-Large-Model/Sana_Sprint_1.6B_1024px_diffusers`；
- `SanaSprintPipeline`；
- 两张 T4 各驻留一份 pipeline；
- bfloat16 pipeline，VAE float32；
- 默认 1024×1024、2 steps、guidance 0；
- WebP quality 90 后 AES-GCM 上传；
- 新旧 cells 并存，部分 v2 路径仍使用旧 GET claim/legacy worker 语义。

迁移前必须补齐 exact Hugging Face revision、依赖版本、输入尺寸约束、输出色彩/格式和真正的 cold/warm/显存基线。

### 3.2 Z-Image-Turbo GGUF

Notebook 当前：

- `stable-diffusion.cpp` 预编译 T4 SM75 release，tag/asset/SHA-256 已固定；
- diffusion `z_image_turbo-Q4_K.gguf`；
- Qwen3-4B-Instruct GGUF text encoder/LLM；
- VAE `ae.safetensors`；
- 两个 T4 各启动一个 `sd-server`；
- 默认 1024×1024、8 steps、CFG 1；
- PNG 转 WebP quality 90，再 AES-GCM 上传；
- worker register/heartbeat/claim/fail/upload 相对完整。

模型文件当前只固定 repo/file 名，没有在仓库契约中固定 revision/hash。Modal 迁移必须先补齐，不可依赖“HF 当前 latest”。SM75 release 也不能直接作为 L40S/SM89 生产产物。

## 4. 当前任务与状态语义

`HubState` 已有值得保留的设计：

- SQLite WAL，多 Uvicorn 进程共享；
- `BEGIN IMMEDIATE` 原子领取；
- queue 按 model 隔离；
- inflight lease、heartbeat renew、过期重排；
- max attempts 后 failed；
- history event ID；
- worker registry/TTL；
- Hub instance ID 防止 Tunnel 指向错误数据库。

迁移映射：

| Kaggle Hub | Modal 栈 |
|---|---|
| queued row | Local Connector Job + Modal FunctionCall accepted |
| worker claim | Modal scheduler；删除自建 claim protocol |
| worker lease | FunctionCall/Workflow stage state；本地只观察 |
| worker registry | capability/deployment health/canary |
| AES upload | Worker 写私有 Volume + result descriptor |
| history | Connector SQLite Job/Event/Artifact store |
| WebSocket result | Connector SSE/event + poll recovery |
| model queue | model-specific Modal App/Worker |

不能同时保留两套调度真值；新系统中 Connector Job 是用户身份，Modal FunctionCall 是远端执行身份。

## 5. Prompt Pipeline 迁移

当前模式 `enhance/creative/translate/clean`、target model adapter、批量单项回退和“回填后用户确认”应保留。

目标位置：统一客户端的 Python Local Connector，而非 `modal-2d` GPU Worker。

理由：

- API key 属于用户本地 Secret；
- prompt 优化不应占 GPU 容器；
- 同一功能可服务 2D、EmbodiedGen text、未来其他 provider；
- 原 prompt/processed prompt/手改/stale adapter provenance 应进入 Job；
- 用户必须确认后才产生可能计费的生成请求。

安全修正：

- OpenAI-compatible key 存操作系统 vault；
- React 不读回已保存 key；
- prompt/error 默认不进入普通日志；
- prompt-injection boundary 保留当前 `SOURCE_PROMPT_START/END` 思路；
- target model adapter 从 capability revision 驱动，不在前端散落。

## 6. Artifact 与画质迁移

Kaggle 当前只保存 WebP quality 90。对于后续 Image→3D，这会引入有损压缩边缘和纹理误差。Modal 2D 首版应：

1. 保存 lossless PNG 或经验证的 lossless WebP 为 primary artifact；
2. 可额外生成较小 preview WebP 供 Gallery；
3. primary descriptor 含 SHA-256、width/height/color mode/model/seed/options；
4. 2D→3D 永远消费 primary，不从 Gallery preview 反向下载；
5. 本地 cache 也按 primary hash 去重。

历史 Kaggle WebP 导入时标记 `legacy-lossy-webp`，仍可用于 3D，但 lineage 必须诚实。

## 7. 安全差距与迁移硬门

当前事实包括：

- 后端默认 token 为 `wangran`；
- Notebook 也有相同 fallback；
- README 记载固定公网 Hub/token；
- 前端把 token 放在 localStorage；
- 同一 token 派生 AES-GCM key；
- WebSocket 无认证；
- `/images`、`/outputs` 是静态公开路径；
- `/upload` 图片读取未像 artifact upload 一样严格先限 encrypted bytes；
- legacy worker owner 校验较宽松。

这些设计只可作为受控原型基线。Modal 产品 Gate 前必须满足：

- 删除所有默认/硬编码生产 token；
- Secret 进 OS vault/Python memory；
- Browser 只用 loopback session token；
- Modal 使用用户 Client/private RPC/Volume；
- artifact 下载 job-scoped、hash-verified；
- 不再通过公网 Tunnel 接收 GPU 结果；
- Kaggle fallback 若保留，使用独立随机 worker credential、轮换和严格 lease owner，不与 Modal credential 共用。

## 8. 历史数据迁移

迁移对象：

- `history`：prompt/source prompt/model/seed/options/time/worker metrics；
- `outputs/<model>`：图片/3D artifact；
- failed records；
- prompt metadata；
- 不迁移 worker session、claim lease 或硬编码 URL/token。

导入流程：

1. 停止写入或创建一致性快照；
2. 读取 SQLite 与文件，核对每个 URL 对应文件；
3. 计算 SHA-256/MIME/dimensions；
4. 写入统一本地 artifact cache；
5. 创建 `provider=kaggle-legacy` 的只读 Job/history；
6. 缺文件记录 warning，不伪造 success；
7. 导入报告统计 total/imported/missing/corrupt/duplicate；
8. 原数据保留到用户确认后再按单独清理计划处理。

## 9. 迁移阶段

### KH-0：冻结与审计

- 固定 SANA/Z-Image canary prompt/seed/output；
- 固定 Hub SQLite/HTTP/self-test；
- 记录 exact notebook、model、release、Kaggle image 状态；
- 立即将默认 token/hardcoded endpoint 记为迁移 blocker；
- 不再给 Hub 添加新正式模型。

### KH-1：供应链迁移

- 在 `modal-build` 构建 SM89 Z-Image runtime；
- 固定 SANA/Z-Image/LLM/VAE revisions/hashes；
- Modal runtime offline/no compile canary。

### KH-2：`modal-2d` shadow

- 同 prompt/seed/options 在 Kaggle 与 Modal 各生成；
- 比较尺寸、格式、像素/感知差异、时长、显存、成本；
- 结果 contract 进入新 artifact manifest；
- 不切客户端默认。

### KH-3：`modal-2d-client` parity

- Prompt Studio、single/batch、Gallery、history、AI prompt pipeline；
- Modal credential/Job/artifact；
- 导入 Kaggle legacy history；
- 用户可显式选择 legacy provider 只读查看。

### KH-4：默认切换与回退窗口

- Modal 成为默认；
- Kaggle 只对 allowlist 可提交；
- 双写禁止，避免重复计费；
- 观察一段完整使用周期的错误/质量/成本；
- 回退必须由用户/运维显式选择。

### KH-5：归档

- 禁止新 Kaggle task；
- 等待 queued/inflight 清空；
- 导出最终历史/failed；
- 撤销 Tunnel/worker token；
- Notebook 标记 research/legacy；
- 仓库 README 指向新三仓计划；
- 保留只读 tag，不删除实验事实。

## 10. 迁移验收

- SANA 与 Z-Image 在 Modal 上各有固定 revision 的真实 canary；
- 同 seed/profile 结果可复现到模型允许的范围；
- lossless primary + preview artifact 可下载和验证；
- Prompt Pipeline provenance 完整且 Secret 不进 React；
- single/batch、restart、cancel、history 达到 Hub 功能等价或更好；
- Kaggle history 可导入并离线打开；
- 2D result 可零歧义交给 3D pipeline；
- TripoSR/其他 3D Notebook 没有被误迁入 `modal-2d`；
- 不再需要 Cloudflare Tunnel/AES 回传才能完成正式生成；
- 默认 token、localStorage token、公开输出路径不进入新系统。

## 11. 非目标

- 修改所有 Kaggle Notebook 以长期双栈运行；
- 把 SQLite worker claim 复制到 Modal；
- 把 Kaggle 双 T4 性能数值当作 Modal SLA；
- 在 2D 迁移中顺手发布全部 3D 实验 Notebook；
- 迁移时删除用户原 `outputs` 或 SQLite。
