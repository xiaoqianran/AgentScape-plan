# `modal-build` 文件级实施计划

## 1. 项目定位

`modal-build` 是 EmbodiedGen 的供应链、运行时适配和 Modal 工作流仓库。目标不是把上游 CLI 原样塞进一个巨大镜像，而是：

- 固定并验证 EmbodiedGen 上游版本；
- 预编译 CUDA 扩展，推理容器只消费 release；
- 用外部 wrapper 将能力拆为可恢复阶段；
- 为每个能力族建立独立 Image/Worker/weights；
- 对外提供统一 capability/job/artifact contract；
- 保证 `EmbodiedGen` 仓库只读。

## 2. CodeGraph 当前事实

### 2.1 构建与供应链

`modal_build/embodiedgen.py` 当前固定：

- EmbodiedGen v2.0.0 commit；
- Python 3.10、CUDA 12.6、Torch 2.8、SM89；
- release tag `embodiedgen-v2.0.0-py310-cu126-torch280-sm89-v1`；
- wheels/torch extensions archive SHA-256；
- release 资产不可覆盖。

生产 runtime 使用 `nvidia/cuda:*runtime*`，显式要求不存在 `nvcc`；nvdiffrast/gsplat 使用 release-only loader。这个约束必须继续作为所有新增 GPU Image 的硬门禁。

### 2.2 当前生产工作流

`runtime/embodiedgen_v2_l40s.py` 已实现：

```text
Image input
→ RembgWorker.prepare（CPU）
→ Sam3DWorker.generate（L40S）
→ MeshWorker.process（CPU，state handoff / simplify / xatlas）
→ lite_gpu_bake（L40S）
→ cpu_finalize（CPU，GLB/OBJ/URDF/validation）

Text input
→ Text2ImageWorker.generate（L40S，固定 Kolors revision）
→ 同一 Image pipeline
```

已有持久资源：

- `modal-3d-embodiedgen-weights`；
- `modal-3d-artifacts`；
- transient state handoff Dict；
- traffic event Dict；
- job state Dict。

已有 API：`POST /jobs`、`POST /text-jobs`、实验性 `POST /jobs/{source_job_id}/retexture`、`GET /jobs/{id}`、`GET /jobs/{id}/files/{name}`，并有周期清理和 `min_cost/cost_first/balanced/burst/auto` autoscale policy。

### 2.3 已形成但尚未生产化的 retexture 纵向切片

最新工作区已超过“只有权重预加载”的阶段，现有事实包括：

- 固定 RoboAssetGen/ControlNet revision 与 `preload_retexture_weights()`；
- 单 L40S、offline load 的 `RetextureWorker`，复用 Kolors base pipeline；
- `POST /jobs/{source_job_id}/retexture`，只接受已成功的 source Job，并异步 `spawn` 派生 Job；
- condition render → 多视图 diffusion → SR/backproject → GLB/OBJ/texture/video 的实际流程；
- 保留 source URDF/GS artifacts，并验证 face count/bounds 以证明 geometry preserved；
- 本地供应链、autoscale、异步端点与结构性校验测试。

这仍只能标为“实验性纵向切片”，而不是生产适配完成：尚缺真实 Modal GPU E2E/cold-warm 证据、独立 capability/version、通用 derived artifact manifest/hash、source retention lease、幂等/cancel/resume，以及与 asset-core 解耦的部署与回滚。

### 2.4 当前主要结构风险

- 生产 runtime 已约 2,000 行，control plane、image、weights、Workers、API、benchmark 混在一起；
- `run_job` 固定顺序串行执行，无法描述多工作流 DAG；
- Job state 没有 contract version、idempotency、cancel/resume、artifact index；
- Dict 中 state 是最终事实但缺事件序列与原子 stage claim；
- 文件下载依赖固定 `RESULT_FILES`，不能支持不同 workflow；
- retry 只能整体重跑或依靠个别 volume fallback；
- Image/Text 结果的 fallback URDF metadata 较保守，不等于完整 sim-ready/affordance；
- 多种可选依赖若继续加入当前 Image，会显著扩大冷启动和供应链风险。

## 3. 目标仓库结构与责任

建议按责任拆分，具体文件名可调整：

```text
modal_build/
  releases/             构建不可变 CUDA/Blender/能力族产物
  supply_chain/         pin、hash、license、SBOM、patch inventory
runtime/
  control_plane.py      capabilities、job、stage、artifact、cleanup
  contracts.py          纯校验/规范化，无 Modal 重依赖
  workflows/
    asset_image_to_3d.py
    asset_text_to_3d.py
    asset_retexture.py
    asset_convert.py
    asset_affordance.py
    scene_background.py
    scene_layout.py
    scene_room.py
  workers/
    asset_core.py
    texture.py
    conversion.py
    affordance.py
    scene.py
    room.py
  deployments/          按能力族创建 Modal App/Image/Volume
patches/
  embodiedgen-v2.0.0/   只存最小、版本化、可验证 patch
tests/
  contracts/ workflows/ supply_chain/ deployments/
```

不要求一次大重构；先抽取纯契约与 control plane，再按工作流迁移，旧 runtime 在兼容窗口内作为 facade。

## 4. 能力族与部署边界

| 能力族 | 建议 App | 主要运行时 | 独立原因 |
|---|---|---|---|
| asset-core | `modal-embodiedgen-asset-v2` | rembg、SAM3D、mesh、bake、text2image | 已验证主链，共享模型状态 |
| retexture | `modal-embodiedgen-texture-v2` | nvdiffrast、多视图 diffusion、SR、bake | 权重/显存与 asset-core 不同 |
| conversion | `modal-embodiedgen-convert-v2` | trimesh、URDF/MJCF/USD 工具 | 大部分 CPU；USD 依赖需隔离 |
| affordance | `modal-embodiedgen-affordance-v2` | segmentation、LLM semantics、grasp、SAPIEN | Secret、GPU 与模拟器依赖不同 |
| scene-background | `modal-embodiedgen-scene-v2` | panorama、Pano2Mesh、gsplat training | 约 30 分钟，资源与 TTL 独立 |
| layout | `modal-embodiedgen-layout-v2` | LLM graph、asset fan-out、composition | 是控制流，不应复制全部模型到一个镜像 |
| room | `modal-embodiedgen-room-v2` | Blender/Infinigen/export | 超大特殊运行时、许可证与启动不同 |
| simulation | `modal-embodiedgen-sim-v2` | SAPIEN/Genesis/可选 Isaac USD | 后置、按 simulator 进一步拆分 |

所有能力族共享 contract，不强制共享 App、Image 或 weights Volume。大型权重 Volume 也按能力族拆，避免每个 Worker 都能看到全部权重。

## 5. EmbodiedGen 只读政策

### 5.0 当前 dirty experiment 的处理

当前 `EmbodiedGen` 工作区存在 `modal_embodiedgen.py`、`modal_postprocess.py` 和若干上游文件修改。它们不属于本计划要继续扩写的正式落点。实施时先：

1. 记录每个修改文件的 base commit、diff、hash、用途和已验证命令；
2. 将可复用的 stage/process-isolation/postprocess 逻辑重新表达为 `modal-build` wrapper 或版本化 patch；
3. 用相同 fixture 对比旧实验与 `modal-build` 生产 Runtime 的输出；
4. 在用户另行确认前，不清理、不回滚、不覆盖这些现有文件；
5. production build 只能从固定上游 commit + `modal-build` patch inventory 重建，不能从 dirty checkout 隐式取文件。

### 5.1 固定上游

每个部署记录：

- upstream repository；
- exact commit/tag；
- submodule commits；
- dependency lock；
- release archive hash；
- patch set ID；
- image digest；
- model weight repo/revision/hash/许可确认。

### 5.2 适配优先级

1. 从 `modal-build` 导入上游公开 Python API。
2. 在 `modal-build/runtime/workflows` 编写 wrapper。
3. 通过环境变量、工作目录和参数控制行为。
4. 只有上游硬编码阻断 headless/release-only 才使用 patch。

### 5.3 Patch 门禁

每个 patch 必须：

- 放在 `patches/<version>/production`；
- 记录目标 commit/文件 hash；
- 解释无法用 wrapper 解决的原因；
- 尽量不改变算法结果；
- 有 `git apply --check`、py_compile 和专属回归；
- 上游升级时默认失效并重新审计；
- 不回写到 `/root/workspace/EmbodiedGen`。

## 6. 工作包

### MB-00：冻结当前验证基线

任务：

1. 记录现有 image/text canary、cold/warm、输出文件和 validation report。
2. 对 `run_job` 每阶段保存固定 fixture/result summary。
3. 记录当前 App、Class/Function、Volume、Dict、Secret、autoscale 配置。
4. 把当前 retexture Worker/Job 纵向切片标为 experimental；在真实 GPU、artifact、恢复和独立部署门禁通过前不发布 production capability。
5. 审计工作区中 EmbodiedGen 的本地改动；计划实施不覆盖、不依赖这些未固定变更。

验收：后续结构迁移可以证明输出、成本和供应链未回退。

### MB-01：纯 Contract Core

从大 runtime 中优先抽取不依赖 GPU/Modal 的纯逻辑：

- workflow/capability 定义；
- request normalization；
- safe job/artifact path；
- option/seed/profile schema；
- job/stage transition；
- artifact descriptor/index；
- error mapping；
- TTL/cleanup decision；
- resource/cost summary；
- lineage/provenance。

要求所有 contract 可在普通 CI 导入，不能因为 import 而初始化 Modal App、下载权重或加载 CUDA 库。

### MB-02：Workflow Registry

建立稳定 workflow ID：

- `embodiedgen.asset.image_to_3d.v1`；
- `embodiedgen.asset.text_to_3d.v1`；
- `embodiedgen.asset.retexture.v1`；
- `embodiedgen.asset.convert.v1`；
- `embodiedgen.asset.affordance.v1`；
- `embodiedgen.scene.background.v1`；
- `embodiedgen.scene.layout.v1`；
- `embodiedgen.scene.room.v1`；
- `embodiedgen.simulation.validate.v1`（后置）。

每个定义包含：输入 schema、option schema、阶段 DAG、必需/可选 artifact、resource class、Secret requirements、expected duration class、TTL、版本/provenance 和 availability。

capability 只能发布已部署并通过 canary 的 workflow；尚未完成的标 `experimental/disabled`，不能让 Connector 提交。

### MB-03：通用 Job/Stage Engine

目标是让 `run_job` 从固定 for-loop 迁为可恢复 stage executor。

核心概念：

- Job：稳定请求、workflow、owner、状态；
- Stage：确定输入、输出、attempt、claim/lease；
- Artifact：不可变 descriptor；
- Event：追加式状态变化；
- Checkpoint：可重新开始的最小边界；
- Worker invocation：资源实现，不持有整体 workflow 真值。

执行规则：

1. Stage 在调用前原子 claim，避免重复调度。
2. 输出写临时目录；验证/hash 后才发布到 artifact index。
3. 相同 stage input hash + revision 可复用成功 checkpoint。
4. retry 增加 attempt，旧 artifact 保留 lineage 但不作为 active output。
5. control plane 崩溃后扫描 running lease，按阶段策略 resume/retry/fail。
6. fan-out/fan-in 支持 layout 多资产；并发数受 policy 控制。
7. stage 只有 `queued/running/succeeded/failed/cancelled/skipped`，Job 另有聚合状态。

### MB-04：Job 持久化、幂等、取消和恢复

当前 Modal Dict 可继续作为首版状态介质，但 schema 必须版本化，并避免多字段非原子覆盖导致丢事件。

任务：

- request hash 与 idempotency key；
- key 同 payload 返回原 Job，不同 payload 冲突；
- event sequence；
- stage attempt/checkpoint；
- `cancel_requested`；
- 可取消 Worker 使用 Modal cancel，不能中断的阶段在边界停止；
- `resume` 只从验证成功 checkpoint 接续；
- active job heartbeat/lease；
- cleanup 绝不把暂时无 heartbeat 的长 GPU stage误删；
- Job status 与 artifact index 分开存但有一致性检查；
- Connector 使用私有 RPC 提交/查询，不依赖公网文件 URL。

### MB-05：Artifact Registry 与目录规范

取代固定 `RESULT_FILES`：

```text
embodiedgen/jobs/<job-id>/
  request.json
  events/
  stages/<stage-id>/<attempt>/
  artifacts/<artifact-id or content path>
  result/manifest.json
```

逻辑 artifact role 示例：

- `source-image`、`canonical-rgba`；
- `visual-glb`、`visual-obj-bundle`；
- `collision-mesh`、`urdf`、`mjcf`、`usd`；
- `texture-basecolor`、`preview-video`；
- `gaussian-splat-ply`、`background-mesh`；
- `layout-json`、`world-bundle`；
- `part-segmentation-glb`、`affordance-json`、`grasp-report`；
- `validation-report`、`lineage-manifest`。

每项包含 hash、bytes、MIME、role、producer stage、relative dependencies、coordinate/unit metadata、validation 和 expiry。OBJ/MTL/texture 必须作为 bundle 表达，不能只下载 OBJ。

### MB-06：现有 Image→3D 迁移

先保持算法与 Worker 资源不变，只换控制契约：

1. `ingest`：验证上传和创建 source artifact。
2. `foreground`：`RembgWorker.prepare`。
3. `reconstruct`：`Sam3DWorker.generate`。
4. `mesh_prepare`：`MeshWorker.process`。
5. `texture_bake`：`lite_gpu_bake`。
6. `finalize`：GLB/OBJ/URDF/preview。
7. `validate`：结构、依赖、geometry、hash、bundle。

专门事项：

- API job 不允许 fallback sample；
- seed 必须从 request 贯穿到 SAM3D；
- state handoff Dict 丢失时只复用同 revision 的 Volume checkpoint；
- fallback URDF 的 `unknown` metadata 明确标为 provisional；
- 当前 collision 如复用 visual mesh，结果不能标“accurate collision”；
- 50k face simplify 与 texture 1024 等 effective options 写入 lineage。

验收：旧 `/jobs` canary 与新 workflow 结果结构、可视几何、文件集合不回退；旧 API 在迁移期转换为新 Job。

### MB-07：现有 Text→3D 产品化

当前 Text2ImageWorker + Image pipeline 已通，不从零重写。补齐：

- capability 明确当前 backend 是固定 Kolors→SAM3D，而非上游所有 backend；
- prompt normalization、语言、长度、seed；
- Text2Image model/revision/license provenance；
- generated image 作为可下载中间 artifact；
- Text2Image 失败可独立 retry；
- 用户可以从生成图继续 Image→3D，不重复计费；
- prompt 内容默认不进日志，只保存到 job-owned artifact；
- 后续如果接 Hunyuan direct text→3D，作为不同 backend/profile，不静默替换。

### MB-08：`asset.retexture` 适配

以上游 `gen_texture.py` 为语义基线，直接产品化当前已存在的 `RetextureWorker + derived Job API`，不另起一条重复实现。

当前已有：固定权重、offline L40S load、source succeeded 门、派生 Job、texture GLB/OBJ/video、source geometry preservation 校验。

阶段建议：

1. ingest mesh bundle：GLB 或自包含 OBJ bundle；
2. mesh validate/normalize：几何、UV、材质、坐标；
3. multi-view render：RGB/normal/position；
4. prompt-conditioned multi-view texture generation；
5. backproject/bake；
6. optional super-resolution；
7. export OBJ/GLB/texture；
8. render preview and validate。

将现有纵向切片补齐为生产能力的任务：

1. 把固定 `RESULT_FILES` 改为该 workflow 自有、版本化 artifact manifest，避免 source/retexture 文件角色混淆。
2. 为输入 source bundle 建立 descriptor/hash/lease；source 被清理时不得留下不可恢复的 derived Job。
3. 记录原 geometry hash、输出 geometry hash、face/bounds tolerance 与验证算法版本；不能只依赖同一进程中的比较结果。
4. 输出继续作为新 derived asset，不覆盖 source；显式 `derived_from` source bundle/artifact hash。
5. 将 delight、IP-adapter、SR、texture resolution 暴露为 capability option 之前，逐项做质量、VRAM和供应链 canary；当前 `delight=false`、`ip_adapter=false` 必须作为 effective options 返回。
6. 增加 idempotency、cancel_requested、stage event、失败恢复；重试 backproject 不应重新执行已经提交成功的前段，除非 artifact 无效。
7. 将 retexture 拆到独立 App/Image/weights/rollback 单元；GPU runtime 保持 offline/no nvcc。
8. 真实 L40S canary 覆盖 cold/warm、不同 topology/UV/材质、损坏 source、源产物过期、OOM与 Volume commit 失败。
9. 大型 condition/multiview 中间产物按 lease/diagnostic policy 清理，失败诊断所需证据先落 manifest。
10. AgentScape 首版只消费 textured GLB、texture 与 provenance；URDF/GS 是沿用 source 的 evidence，不能误报为本次 retexture 重新验证。

关键门禁：

- 不允许 OBJ 引用逃逸 bundle；
- 无 UV 时明确是自动 unwrap 还是拒绝；
- 保持原 mesh topology/transform 的策略可配置并记录；
- prompt 与 mesh count 一一对应；
- texture resolution 有 Web budget profile；
- 输出 GLB hash、材质/texture summary；
- 上游 `entrypoint()` 当前是 CLI 风格时用 wrapper 调用内部能力，确实无法稳定调用再做最小 patch。

### MB-09：`asset.convert` 适配

以上游 `embodied_gen/data/asset_converter.py` 和 URDF converter 为基线，优先 CPU 能力：

- EmbodiedGen URDF bundle → MJCF；
- mesh/URDF → USD/USD physics variant；
- 输入 GLB/OBJ → normalized GLB + conservative URDF bundle；
- bundle dependency rewrite/validation。

先服务 AgentScape 的真实需求：GLB、collision evidence、unit/axis/scale 与 URDF metadata。Isaac-specific Physics USD 依赖单独 capability，不能让普通 conversion Image 携带完整 Isaac Sim。

验收不能只检查文件存在：解析目标格式、所有引用存在、单位/坐标明确、mesh count 非零、转换往返不会出现数量级尺度漂移。

### MB-10：`asset.affordance` 适配

> 2026-08-24 实时状态：native CUDA wheels 已在 L40S/SM89 实跑；P3-SAM 已对真实 production Job 完成 50,000 faces / 4 parts E2E；GraspGen Franka 权重已固定 revision 与 SHA 并预加载。当前未完成项是 raw GraspGen inference、GPT semantic、SAPIEN evaluation、统一 Job/API，以及 AgentScape compiler-native face-label bridge。执行细节与证据以 [`04-live-execution-state.md`](./04-live-execution-state.md) 为准。

按上游依赖拆阶段：

1. URDF/bundle ingest；
2. functional part segmentation；
3. part semantic annotation；
4. grasp candidate generation；
5. SAPIEN grasp evaluation；
6. immutable enriched bundle finalize。

原则：

- 上游会原地修改 URDF；Modal wrapper 必须在 job 副本上运行，原输入不可变；
- LLM Secret 仅进入 semantic worker；
- segmentation、semantic、grasp proposal、sim-validated grasp 的 evidence level 分开；
- `--no-run-*` 映射成 checkpoint reuse，而非任意跳过必需依赖；
- `mesh_part_seg.glb` 的 face IDs、`affordance_annot.json` schema、更新后 URDF 均入 bundle；
- 对 AgentScape 首先提供 part evidence，不自动把 semantic label 升为 open/close joint。

### MB-11：`scene.background` 适配

以上游 `gen_scene3d.py` 为基线，阶段：

1. prompt → panorama；
2. panorama quality check；
3. Pano2Mesh；
4. 3DGS training（默认上游 4000 steps）；
5. mesh/PLY/config/video export；
6. budget/quality validation。

因为典型约 30 分钟：

- 必须有 heartbeat、长 TTL、阶段 checkpoint 和明确成本预估；
- panorama/mesh 成功后 3DGS 失败可单独重试；
- AgentScape 首版只消费 `mesh_model` 转换后的 GLB；
- `gs_model.ply` 与 `gsplat_cfg.yml` 保留但 capability 标明浏览器 runtime 不直接支持；
- max_steps 等高成本 option 有上限和 profile，不直接开放任意数值。

### MB-12：`scene.layout` 适配

Layout 是控制工作流，不应该让一个 L40S 容器串行生成全部对象。阶段：

1. task description → scene graph proposal；
2. graph/schema validation；
3. background resolve/generate；
4. asset resolve-first，再对缺失项 fan-out Text→3D；
5. per-asset validation/admission evidence；
6. deterministic composition/robot pose；
7. layout.json/world bundle；
8. optional SAPIEN preview；
9. validation。

要求：

- 子 Job 有父子关系与去重；
- 失败对象不让整个 30 分钟任务变成不可诊断错误；
- layout.json 是 provider proposal，进入 AgentScape 后仍需转 WorldSpec 和 Runtime validation；
- 背景、asset、images、scene tree、preview 全有独立 artifact；
- 坐标、单位、旋转、scale、robot/entity 语义写入 bundle manifest。

### MB-13：`scene.room` 适配

Room/Infinigen/Blender 是独立立项：

- 构建固定 Blender/Infinigen runtime，记录许可和 image size；
- natural-language GPT router 与 explicit room config 分开；
- `generate`、`export_urdf`、`export_usd` 拆 checkpoint；
- 原始 Blender scene 是中间 artifact，可用于 re-export；
- URDF whole-room 与 per-instance mesh bundle；
- USD 默认没有 physics 时必须明确 provisional；
- House/large-scene 有独立 timeout、磁盘和预算 profile；
- 实例 GLB 化和 AgentScape environment bundle 是额外 adapter stage。

在没有测得冷启动、磁盘峰值、生成时长和 Modal 对 Blender/子进程兼容前，不进入默认 capability。

### MB-14：仿真与 Soft-body 后置能力

拆成独立可选 workflow：

- SAPIEN load/render/physics validation；
- parallel simulation；
- Genesis cloth conversion/simulation；
- grasp quality evaluation；
- Isaac USD physics postprocess（若运行环境可行）。

它们不是 AgentScape 首次接通 Modal 的依赖。只有当资产/世界 bundle 稳定，并明确下游消费结果时才投入。

### MB-15：Capabilities 与 Connector RPC

统一 `embodiedgen.capabilities.v1`：

- workflows、versions、status；
- input/output artifact roles；
- option schema/profiles；
- duration/cost/resource class；
- Secret/license prerequisites；
- stage names；
- cancellation/resume support；
- result TTL；
- AgentScape consumption notes。

提供私有 Modal RPC：submit/get/list/cancel/get-artifact-manifest；现有 proxy-auth ASGI 保持兼容，但 Local Connector 优先走用户 Modal Client，不要求浏览器处理 proxy auth。

### MB-16：资源、成本与并发政策

每个 stage 声明：CPU/GPU/内存/磁盘/timeout、weight volume、scaledown profile、max concurrency。

初始政策：

- GPU worker 默认 `max_containers=1`；
- 不使用 container input concurrency；
- CPU weight preload/sync `max_containers=1`；
- layout fan-out 是 control-plane 并发，不等于无限 GPU 并发；
- auto profile 只在有流量证据时提高 warm window；
- 成本记录区分 execution 与 idle tail；
- scene/room 提交前给出 duration/cost class 确认；
- 新 workflow 先 canary/allowlist，再进入普通 capability。

### MB-17：可观测性

统一字段：workflow/job/parent job/stage/attempt、request hash、artifact hash、worker revision、container/cold-warm、queue/load/run/commit、GPU peak、input/output bytes、cost estimate、error code。

必须能回答：

- Job 卡在哪个 stage；
- 此 stage 是否安全重试；
- 使用哪个上游/model/release/patch；
- 哪些 artifact 已验证；
- 是否因 control plane、模型、Volume、Secret 或输入失败；
- 清理任务为何删除/保留某个 Job。

### MB-18：测试体系

**供应链**

- exact commits/revisions/hashes；
- release immutable；
- patch applies only to target；
- runtime image no `nvcc`；
- offline model load；
- SBOM/license inventory。

**纯契约**

- workflow schemas；
- state transition；
- path/bundle safety；
- artifact index/hash；
- idempotency；
- TTL/lease/cleanup；
- error normalization。

**工作流 mock**

- stage success/fail/retry/resume/cancel；
- checkpoint reuse revision mismatch；
- fan-out/fan-in partial failure；
- artifact publish/commit failure；
- control process restart。

**真实 canary**

- Image→3D、Text→3D；
- retexture demo meshes；
- convert demo bundle；
- affordance demo URDF；
- scene tiny/standard profile；
- layout 单任务最小 world；
- room minimalist explicit config；
- 每条都解析并独立验证结果。

**AgentScape fixture**

- GLB + URDF + collision + validation bundle；
- part segmentation/affordance evidence；
- background mesh bundle；
- layout world bundle；
- 坐标/单位和 malformed cases。

### MB-19：部署、兼容与回滚

部署顺序：

1. contract/control plane；
2. 新 capability disabled；
3. weights preload 与 image build；
4. Worker direct canary；
5. workflow canary；
6. Connector contract test；
7. allowlist；
8. general availability。

每个能力族独立回滚；capability 能将单 workflow 标 degraded/disabled。旧 Job 必须继续由其提交 revision 查询和下载，不能因为发布新版本丢失。

## 7. 实施顺序

1. MB-00～05：先建立内核，不扩功能。
2. MB-06～07：迁移现有 Image/Text，证明不回退。
3. MB-08：把已存在的实验性 retexture 纵向切片产品化并通过独立 Gate。
4. MB-09：conversion，为 AgentScape bundle 服务。
5. MB-10：affordance/part evidence。
6. MB-11：background scene。
7. MB-12：layout。
8. MB-13：room。
9. MB-14：simulation/soft-body。

不得跳过现有工作流迁移，直接继续往大 runtime 添加所有功能。

## 8. 完成定义

- `EmbodiedGen` 工作区没有因 Modal 适配产生提交或文件改动；
- 每个能力都有固定上游、release、weights 和 patch provenance；
- Image/Text 主链迁移到通用 workflow contract 且不回退；
- retexture/convert/affordance/scene/layout/room 按 gate 独立上线；
- Job 可幂等、取消、阶段重试和恢复；
- artifact 是 versioned manifest，不再靠固定文件名猜；
- AgentScape 所需 bundle 有坐标、单位、collision/part evidence；
- GPU runtime 不编译、不运行时下载；
- 新能力不会扩大所有既有 Worker 镜像；
- 每条已启用 workflow 都有真实 canary、成本基线和回滚方案。

## 9. 非目标

- 一次性适配 EmbodiedGen 所有研究脚本；
- 让 Modal runtime 成为 EmbodiedGen fork；
- 在一个万能镜像安装 Blender、SAPIEN、Genesis、Isaac 和全部生成模型；
- 把 LLM/视觉语义直接当作已物理验证事实；
- 在没有 AgentScape 3DGS renderer 前优先开发浏览器 3DGS 输出链。
