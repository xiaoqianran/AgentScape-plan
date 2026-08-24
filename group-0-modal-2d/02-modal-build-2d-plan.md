# 共享 `modal-build` 的 2D 构建计划

## 1. 定位

同一个 `modal-build` 仓库负责 2D 与 EmbodiedGen/3D 的“重型构建事实”，但 release、环境矩阵、patch 和测试按能力隔离。2D 新增内容只服务：

- SANA Sprint 1.6B Diffusers runtime 的精确环境；
- Z-Image-Turbo GGUF 的 `stable-diffusion.cpp` SM89 release；
- 必需的通用 CUDA/图像编码产物。

模型 Worker、Gateway 和 API 仍放 `modal-2d`，本地 UI/Prompt Pipeline 放 `modal-2d-client`。

## 2. 构建矩阵

| Target | 构建类型 | 初始目标 | 关键产物 |
|---|---|---|---|
| SANA | dependency/image lock | Modal 选定 GPU/Python/CUDA/Torch | env manifest、lock、compatibility canary |
| Z-Image | native CUDA release | L40S/SM89 初始 | `sd-server`、shared libs、build manifest、archive hash |
| Z-Image models | immutable weight manifest | exact HF revisions | diffusion GGUF、Qwen GGUF、VAE hashes |
| image codec | 尽量系统包/固定 wheel | CPU/GPU image | PNG/WebP encode/decode canary |

如果未来保留 Kaggle T4 fallback，SM75 是独立 target/release，不能让一个无架构标识的 archive 同时代表 SM75 与 SM89。

## 3. Release 命名

命名必须包含：component、upstream commit、Python/CUDA/Torch 或 runtime ABI、GPU arch、revision。例如概念上：

```text
stable-diffusion-cpp-<commit>-cu<version>-sm89-v1
modal2d-sana-<model-revision>-py<version>-cu<version>-torch<version>-sm89-v1
```

release manifest 包含：

- upstream repository/commit；
- build command/CMake flags；
- base image digest；
- compiler/CUDA versions；
- target architectures；
- archive file list/SHA-256；
- runtime dependency list；
- canary hardware/result；
- license/SBOM；
- created by/source revision。

不可覆盖已有 tag；变更任何产物必须 bump tag。

## 4. SANA 供应链

### MB2D-SANA-01：模型固定

- 固定 HF repo exact commit，不使用浮动 snapshot；
- 记录 model index 及所有 weight/config hash；
- 记录模型/数据许可；
- CPU preload 到专属 weights Volume；
- revision marker 与完整文件验证；
- 下载函数拥有网络，GPU Worker offline。

### MB2D-SANA-02：环境固定

- Python、Torch、CUDA、Diffusers、Transformers、Accelerate、xformers/attention backend；
- bfloat16 在目标 GPU 的支持与质量；
- VAE float32 的必要性用 canary 固定；
- 不依赖 Kaggle 当前预装包；
- import/model load/generate 不触发 JIT 编译或网络。

### MB2D-SANA-03：行为 canary

固定 prompt/seed/size/2-step：

- 输出可解码、尺寸/色彩正确；
- cold/warm load/推理/VRAM；
- 256/512/1024 与边界尺寸；
- 非英文 prompt 在不经过 Prompt Pipeline 时的行为；
- 超限/错误 option；
- deterministic tolerance/hash strategy。

SANA 没有自定义 native extension 时不为“结构对称”制造无意义 release archive；环境 manifest 与 immutable model revision就足够。

## 5. Z-Image 供应链

### MB2D-ZI-01：上游审计

- 固定 `stable-diffusion.cpp` exact commit；
- 对账当前 Kaggle tag `sdcpp-t4-sm75-de298c225bed` 的源码 commit、patch、flags；
- 列出 `sd-server` 运行所需 `.so`；
- 明确 `--diffusion-fa` 等 feature flags；
- 审计 HTTP API schema 与退出/并发语义。

### MB2D-ZI-02：SM89 build

- 使用 devel image 在 build job 编译；
- target SM89，保留必要 PTX 与否由兼容测试决定；
- release archive只含运行时必需文件；
- RPATH/LD_LIBRARY_PATH 策略固定；
- 在干净 runtime image 解压运行；
- runtime image不得有 nvcc/CMake/source tree；
- `sd-server /v1/models` 与 txt2img canary。

### MB2D-ZI-03：模型文件固定

分别固定：

- `leejet/Z-Image-Turbo-GGUF` diffusion file/revision/hash；
- `unsloth/Qwen3-4B-Instruct-2507-GGUF` file/revision/hash；
- `Comfy-Org/z_image_turbo` VAE file/revision/hash；
- 每项许可与组合分发限制。

CPU preload 按 manifest 下载并在 commit 前验证；GPU `sd-server` 仅 local path。

### MB2D-ZI-04：服务进程策略验证

Kaggle 当前每 GPU 一个长期 `sd-server`。Modal 首版建议一个容器一个 GPU/一个 server，避免在同容器自行管理两卡。需要验证：

- Modal lifecycle 启动/健康/终止子进程；
- port 只绑定 loopback；
- 每次请求超时/取消时 server 状态；
- crash 后 container replacement；
- stdout/stderr 日志截断与脱敏；
- 一次只处理一个输入，直到并发安全有证据。

## 6. 建议文件责任

```text
modal_build/
  sana.py                         环境/model manifest检查或辅助构建
  stable_diffusion_cpp.py         原生 SM89 build + immutable release
envs/
  sana-...-sm89.json
  z-image-...-sm89.json
patches/
  stable-diffusion-cpp-<commit>/  仅在确有必要时
tests/
  test_sana_supply_chain.py
  test_zimage_supply_chain.py
```

EmbodiedGen release/tests 不与 2D 混在同一 target 函数里；共享的 hash/release helper 可以复用纯逻辑。

## 7. 构建工作包

### MB2D-00：现有 Kaggle 产物盘点

- 获取所有 release tag/asset/hash/source commit；
- 确认哪些由 `kaggle-build` 生成，哪些只有 Notebook 文档；
- 不把未知来源二进制复制进 `modal-build`；
- 保留 SM75 历史 manifest供回归。

### MB2D-01：环境与模型 manifest schema

统一字段：component/upstream/revision/runtime/CUDA/GPU arch/files/hashes/license/canary。由 `modal-2d` Image 在构建时校验兼容范围。

### MB2D-02：SANA pin/offline proof

完成 exact model/dependency pin、CPU preload、offline load/generate、质量 fixture。

### MB2D-03：Z-Image SM89 release

完成 native build、archive/hash、immutable publish、干净 runtime canary。

### MB2D-04：权重 preload contract

为每个模型定义 marker、expected files/hash、原子同步、失败清理、Volume revision；避免半下载权重被 Worker 当作 ready。

### MB2D-05：SBOM 与许可门

记录模型、GGUF、Qwen、VAE、Diffusers、stable-diffusion.cpp 许可；明确 release 是否可再分发，权重是否只在用户 workspace 下载。

### MB2D-06：CI/Release

- source pin/hash tests；
- patch apply test；
- archive no-clobber；
- runtime no compiler；
- ABI/file manifest；
- GPU canary由受控部署任务执行；
- release失败不留下看似完整 tag/asset。

## 8. 验收

- SANA 与 Z-Image 所有模型/代码来源 exact pin；
- Z-Image 有 SM89 release，不复用 SM75 假兼容；
- archive SHA-256 和 manifest 可独立验证；
- GPU runtime无 nvcc/CMake/在线下载；
- CPU preload后离线 load/generate；
- 一个模型的 release升级不影响 EmbodiedGen/Hermit/Pixal产物；
- license/SBOM 和回滚 tag完整；
- `modal-2d` 只通过 versioned release/manifest消费构建产物。

## 9. 非目标

- 在 `modal-build` 承载 Gateway/UI/Job history；
- 继续以 Kaggle 预装环境作为生产锁文件；
- 每次启动 Modal GPU 编译 `stable-diffusion.cpp`；
- 为所有潜在 GPU 架构一次性构建未验证二进制；
- 在同一 release tag 覆盖修复文件。
