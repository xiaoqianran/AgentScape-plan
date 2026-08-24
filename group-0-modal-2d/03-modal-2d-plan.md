# `modal-2d` 新仓实施计划

## 1. 项目定位

`modal-2d` 是与 `modal-3D` 对称但不互相依赖的云端 2D 模型仓库。首批只包含两条 Text→Image Worker：

- SANA Sprint 1.6B；
- Z-Image-Turbo GGUF。

它负责 capability、CPU Gateway、独立 Worker/Image/weights、统一 Image Artifact；不负责 Prompt AI、Tauri、凭据、Gallery、3D 编排或 AgentScape。

## 2. 目标结构

```text
modal_2d/
  common.py               model IDs、result helper
  contracts.py            request/options/artifact/error纯校验
  capabilities.py         唯一能力清单
  gateway.py              CPU async submit/status metadata
  artifacts.py            safe path/hash/image validation
  sana_sprint.py           独立 App/Image/weights/Worker
  z_image_turbo.py         独立 App/Image/weights/Worker
  operations.py           preload/canary/cleanup
tests/
docs/
```

具体命名可调整，但不能把两个模型塞进同一重量镜像，也不能复制 Kaggle Hub 的 worker claim/lease控制面。

## 3. 部署边界

| 模型 | App | Runtime | Weight Volume | 初始并发 |
|---|---|---|---|---|
| SANA | `modal-2d-sana-sprint` | Diffusers resident pipeline | `modal-2d-sana-weights` | GPU `max_containers=1` |
| Z-Image | `modal-2d-z-image-turbo` | release-only `sd-server` wrapper | `modal-2d-zimage-weights` | GPU `max_containers=1` |
| Gateway | `modal-2d-gateway` | slim CPU | 无模型权重 | 轻量路由 |

Artifacts 首选进入中性、可跨 2D/3D 的内容寻址 Volume。若第一阶段必须沿用既有 `modal-3d-artifacts`，只能作为兼容部署名；contract 中不得出现“3D专属”语义。Gate U0 前确定新 `modal-creative-artifacts` 或等价中性 Volume 的迁移方案。

## 4. Capability Contract

`modal-2d.capabilities.v1` 包含：

- service/deployment/contract version；
- model stable ID/status/revision；
- input=`text` 与 prompt limits/language notes；
- output primary/preview formats；
- width/height/aspect/pixel limits及倍数约束；
- seed range；
- profiles/default/effective options；
- negative prompt支持；
- duration/resource class；
- model/release/weights provenance；
- deprecation/compatibility；
- cross-pipeline hint：primary image适合 Image→3D。

初始 profile 以 Kaggle事实为 canary起点，但必须在 Modal实测后发布：

| Model | 初始候选 profile | 当前原型参数 | 发布前必须验证 |
|---|---|---|---|
| SANA | `recommended` | 1024²、2 steps、guidance 0 | 尺寸范围、bf16、VAE float32、质量 |
| Z-Image | `recommended` | 1024²、8 steps、CFG 1 | GGUF/runtime、尺寸倍数、server稳定性 |

不能仅因为旧 Hub 允许 64～4096 就在云端 capability 宣称全部尺寸受支持。

## 5. Request Contract

必需：model、prompt、profile。可选：seed、width、height、negative prompt、explicit schema options、idempotency key、retention。

校验顺序：

1. contract/model/status；
2. prompt type/length/control chars；
3. profile/options；
4. width/height/pixels/aspect/alignment；
5. seed；
6. idempotency；
7. spawn。

Prompt 原文属于 Job lineage，但默认不进普通日志。Gateway 返回 request/call/model/deployment/effective options，不返回模型内部 Secret/path。

## 6. Output Contract

每次成功至少产生：

- lossless primary image：优先 PNG；
- preview WebP/JPEG：可选、明确有损；
- JSON result manifest。

descriptor 包含：

- artifact ID/role/path/MIME/bytes/SHA-256；
- width/height/color mode/alpha/ICC策略；
- model/revision/release/seed/effective options；
- prompt hash和受保护的prompt lineage reference；
- load/inference/encode/commit timing；
- GPU/VRAM；
- validation/warnings/created/expires。

primary 在 Volume commit/hash/解码验证后才返回。preview 失败可成为 warning，不应毁掉有效 primary。

## 7. SANA Worker 计划

### M2D-SANA-01：Image/weights

- 使用 `modal-build` 固定环境/model manifest；
- CPU preload/sync exact revision；
- GPU offline/local_files_only；
- `@modal.enter` 只加载一次 resident pipeline；
- VAE dtype策略记录并 canary；
- 工作目录无跨请求污染。

### M2D-SANA-02：Inference

- strict prompt/options；
- per-request CUDA generator/seed；
- `torch.inference_mode`；
- 明确 guidance/steps/size；
- CUDA同步用于准确计时；
- 失败清理 tensor/cache但不卸载模型；
- 输出 PIL/array转 lossless primary。

### M2D-SANA-03：特定测试

- 2-step推荐配置；
- empty/超长prompt；
- 不合法尺寸；
- 固定seed重复；
- cold/warm；
- VAE精度回归；
- OOM后的容器健康。

## 8. Z-Image Worker 计划

### M2D-ZI-01：Image/weights

- runtime image只解压 `modal-build` SM89 release；
- exact diffusion/LLM/VAE Volume；
- `sd-server` 绑定容器 loopback随机/固定内部端口；
- lifecycle负责启动、health、日志、退出；
- 不开放公网 server端口。

### M2D-ZI-02：Inference wrapper

- 将统一 request映射到 `/sdapi/v1/txt2img`；
- response base64/PNG严格检查；
- timeout/cancel/container crash稳定错误；
- server HTTP 500与无images分开；
- 每容器一次只发一个请求；
- 请求完成后检查子进程仍健康。

### M2D-ZI-03：特定测试

- release/hash/revision marker；
- server crash/restart；
- malformed JSON/base64/PNG；
- 8-step推荐配置；
- fixed seed；
- OOM/timeout后容器策略；
- runtime无编译器/在线下载。

## 9. Gateway 与幂等

`modal-2d-gateway`：

- capability-driven route；
- CPU option校验；
- stable request ID/idempotency；
- `spawn`远端 worker；
- 返回 FunctionCall ID；
- 相同 key+payload复用，相同 key+不同payload冲突；
- model disabled时提交前拒绝；
- legacy Kaggle model alias仅在迁移窗口归一化，不永久暴露。

Batch 不在单个 GPU call 内循环几十张。Client 创建 parent batch和多个 child Job；云端按正常队列/并发政策调度，避免一次失败丢整批。

## 10. 2D→3D Cloud Handoff

目标是避免“下载 Gallery preview再上传”：

1. 2D primary image拥有中性 artifact descriptor/hash；
2. 3D Gateway允许受控 `modal-2d/jobs/...` 或内容寻址 namespace；
3. Connector创建 downstream request，引用 primary artifact；
4. 若两个部署暂不共享 Volume，Connector用lossless本地cache按hash桥接上传；
5. downstream lineage记录 `derived_from` 2D artifact ID/hash；
6. 可选 Cloud SAM/canonical materialization形成新的中间 artifact；
7. preview artifact永远不可作为默认 downstream input。

共享 Volume 是传输优化，不是身份耦合；最终身份仍是 descriptor/hash。

## 11. Prompt AI 边界

`modal-2d` 不调用 OpenAI-compatible Prompt Pipeline。它只接收最终确认 prompt和可选 provenance summary。理由：Secret/隐私/用户确认/多provider复用均属于本地 Connector。

模型专属 prompt建议可在 capability提供静态 hints，但不能云端静默改用户文本。

## 12. 运维与成本

- GPU默认 `min_containers=0/max_containers=1`；
- 不启用 container input concurrency；
- weights sync CPU/max=1；
- cold/warm/load/inference/encode/commit分开；
- Z-Image子进程load计入cold；
- output bytes/preview节省与PNG成本都记录；
- warm window只有流量证据后调整；
- 单模型可 degraded/disabled/rollback；
- 模型发布不要求同时重发另一个模型。

## 13. 测试

**单元**

- capability/schema；
- prompt/options/size/seed；
- idempotency；
- artifact path/hash/image decode；
- error mapping；
- model route uniqueness。

**Mock Modal**

- Gateway spawn/result/error；
- Volume commit失败；
- disabled model；
- duplicate submit/cancel race。

**GPU canary**

- 每模型固定prompt/seed推荐profile；
- lossless primary和preview；
- cold/warm；
- offline/no compile；
- OOM/timeout/invalidinput；
- output independent decoder。

**Cross-stack**

- `modal-2d-client` capability/job/download；
- 2D primary→SAM→modal-3D；
- unified client artifact cache/hash；
- AgentScape composed route fixture。

## 14. 部署顺序

1. contracts/capabilities；
2. weights preload；
3. SANA direct canary；
4. Z-Image direct canary；
5. Gateway；
6. client contract tests；
7. Kaggle shadow comparison；
8. disabled→allowlist→available；
9. 2D→3D handoff。

## 15. 完成定义

- 两模型独立 App/Image/weights/revision；
- capability取代Hub硬编码事实；
- GPU runtime offline/no compile；
- request/options在CPU层校验；
- lossless primary、preview、manifest/hash可验证；
- Job async/idempotent/cancellable并可被client恢复；
- 2D primary可直接成为3D输入并保留lineage；
- Prompt Pipeline/Secret不进GPU仓；
- 正向/负向/cold/warm/cross-stack canary齐全；
- 无Kaggle Tunnel/worker claim/AES upload依赖。

## 16. 非目标

- 首版迁移TripoSR或其他3D Notebook；
- 复制Kaggle双T4 dispatcher；
- 在Gallery只存有损图；
- 暴露匿名公网Text→Image API；
- 首版支持任意LoRA/ControlNet/inpainting/video；
- 在云端自动改写用户prompt。
