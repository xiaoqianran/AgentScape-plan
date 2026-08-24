# `EmbodiedGen` 只读阶段地图与 Modal 适配清单

## 1. 使用原则

本文只解释上游能力、调用边界和适配优先级。`/root/workspace/EmbodiedGen` 是只读参考源；任何 Modal wrapper、patch、测试 fixture 或部署代码都应放在 `modal-build`。

计划编写时的工作区并非纯上游状态：存在未提交的 `modal_embodiedgen.py`、`embodied_gen/scripts/modal_postprocess.py`，以及 `imageto3d.py`、`backproject_v3.py`、`gs_model.py` 修改。这些内容可作为过去 Modal/headless 调试与 stage 隔离的证据，但不改变“EmbodiedGen 不修改”的目标边界：实施前先登记 diff/hash，提取可复用思想到 `modal-build` wrapper/版本化 patch；不覆盖或删除用户现有改动，也不让生产部署依赖这份 dirty checkout。

状态说明：

- **已适配**：`modal-build` 当前已有真实生产链。
- **部分适配**：已有 pin/preload/某些 stage，但无完整 workflow。
- **待适配**：上游能力存在，Modal 尚无产品级 workflow。
- **后置**：不是 AgentScape 前几阶段的必要依赖。

## 2. 总能力图

```text
Image ─────────────┐
                   ├→ Image→3D ─→ mesh / GLB / GS / URDF / preview
Text → Text→Image ─┘                   │
                                      ├→ Retexture
                                      ├→ Convert to MJCF/USD
                                      └→ Affordance
                                           ├→ part segmentation
                                           ├→ semantics
                                           └→ grasp + simulation

Text → Panorama → Mesh → 3DGS ─────────────→ Background Scene
Task → Scene Graph → many Text→3D + Background → Layout World
Prompt/config → Infinigen/Blender → URDF/USD ─→ Room/House

Asset/World → SAPIEN/Genesis/other simulator → validation/training（后置）
```

## 3. 阶段总表

| 能力 | 上游主要入口 | 主要输入 | 核心输出 | 当前 Modal 状态 | AgentScape 优先级 |
|---|---|---|---|---|---|
| Image→3D | `scripts/imageto3d.py` | image、backend、retry/options | GLB/OBJ/texture/collision/GS/URDF/video | 已适配 SAM3D 路径 | P0 |
| Text→3D | `scripts/textto3d.py` | prompt、backend、seeds/retries | generated image + Image→3D bundle | 已适配 Kolors→SAM3D 路径 | P0 |
| Texture | `scripts/gen_texture.py` | mesh + prompt + seed | textured GLB/OBJ/texture/multiview/video | 部分适配：实验性 Worker/派生 Job，生产门未过 | P1 |
| Convert | `data/asset_converter.py` | URDF/mesh + target simulator | MJCF/USD/URDF variants | 待适配 | P1 |
| Affordance | `scripts/affordance_annot/*` | URDF visual/collision bundle | part GLB、semantic/grasp JSON、updated URDF | 待适配 | P1 |
| Background scene | `scripts/gen_scene3d.py` | prompt、seed、GS config | pano、mesh PLY、GS PLY/config、video | 待适配 | P2 |
| Layout | `scripts/gen_layout.py` + `compose_layout.py` | task、background list、retry/robot | layout.json、多资产、background、preview | 待适配 | P2 |
| Room/house | `scripts/room_gen/gen_room.py` | prompt/config、seed、complexity | Blender、room URDF/per-instance mesh、USD/textures | 待适配 | P3 |
| Simulation | `simulate_sapien.py`、`parallel_sim.py` | layout/URDF | render、physics/task results | 后置 | P4 |
| Soft body | `deformable_sim/*` | generated asset + material | cloth asset、Genesis sim/report | 后置 | P4 |

P0/P1/P2/P3 是实现顺序，不是对研究价值的评价。

## 4. Image→3D 详细映射

### 4.1 上游行为

`embodied_gen/scripts/imageto3d.py` 通过 `_build_image3d_pipeline()` 选择后端，`entrypoint()` 处理批量输入、retry 与最终输出。上游支持：

- SAM3D（本地）；
- TRELLIS（本地）；
- HUNYUAN3D（Tencent Cloud）。

语义阶段：

1. foreground removal/conditioning；
2. backend reconstruction；
3. quality check/retry；
4. Gaussian + mesh state；
5. render/backprojection/texture；
6. visual/collision/export；
7. URDF/metadata/video。

典型结果：`mesh/*.glb`、OBJ/MTL/texture、collision mesh、3DGS PLY、URDF、video。

### 4.2 当前 Modal 覆盖

当前 `modal-build` 覆盖 SAM3D 主路径，并按 CPU/L40S/CPU/L40S/CPU 拆阶段。尚未覆盖：

- TRELLIS backend；
- HUNYUAN3D backend；
- 上游完整 quality checker/retry 策略；
- 精确 collision 生成的可验证声明；
- 完整语义 metadata。

### 4.3 计划输入

- image artifact；
- backend=`sam3d` 首版；
- seed；
- quality/profile；
- optional foreground policy；
- output budget profile；
- idempotency/retention。

### 4.4 计划输出与准入

必需：visual GLB、URDF、validation、lineage。可选：OBJ bundle、collision、GS PLY、video。必须分别标注 evidence：

- GLB structural verified；
- collision 是独立生成还是 visual fallback；
- URDF parse verified；
- physics metadata 是 measured/default/unknown；
- GS PLY retained but AgentScape unsupported。

### 4.5 后端扩展政策

TRELLIS 与 HUNYUAN 不作为相同 workflow 的隐式 fallback。它们应是 capability 中显式 backend/profile，并分别有 Secret、成本、许可、结果差异和 canary。

## 5. Text→3D 详细映射

### 5.1 上游行为

`scripts/textto3d.py` 的 `text_to_3d()` 支持：

- SAM3D/TRELLIS：先 Text→Image，再进入 Image→3D；
- HUNYUAN3D：prompt 直接交云服务，跳过 Text→Image；
- image/asset/pipeline 多级 retry 与 quality check。

### 5.2 当前 Modal 覆盖

当前覆盖固定 Kolors Text→Image revision，然后复用 SAM3D pipeline。它是有效的 Text→3D，但不等于支持上游全部 backend/retry。

### 5.3 计划阶段与 checkpoint

1. prompt normalize；
2. text image generation；
3. generated image validation；
4. optional candidate/quality decision；
5. Image→3D child workflow；
6. combined finalize。

Text→Image 结果必须持久化，后续 reconstruction 失败时无需重新生成图片。用户可从生成图片 fork 一个新的 Image→3D Job。

### 5.4 AgentScape 使用方式

AgentScape 的 `generateAsset(prompt)` 首版映射此 workflow；结果仍必须进入 Asset Compiler/Admission。prompt category、movable、affordance 等只能作为 intent/provenance，不直接成为 runtime physics 事实。

## 6. Texture 详细映射

### 6.1 上游行为

`scripts/gen_texture.py` 使用 `TextureGenConfig`，对 mesh：

1. 渲染 multi-view RGB/normal/position；
2. prompt-conditioned texture generation；
3. backprojection/bake；
4. 输出 textured OBJ/GLB、material/texture；
5. 输出 `color.mp4`。

输入通常是一个或多个 mesh path 与一一对应 prompt。

### 6.2 当前 `modal-build` 实现事实

最新工作区已经用单个 L40S `RetextureWorker` 跑通 condition render、多视图 Kolors ControlNet、SR/backproject、GLB/OBJ/video export，并通过派生 Job API 复用成功的 source Job。当前实现固定 `delight=false`、`ip_adapter=false`，复制 source URDF/GS，并用 face count 与 bounds tolerance 检查 geometry preserved。

这是一条有价值的实验性纵向切片，但尚未证明：真实 Modal GPU canary、不同 mesh/UV 的鲁棒性、source retention、每个输出 hash/role、失败阶段恢复、独立部署与 AgentScape evidence 语义。因此下面的拆分仍是产品化目标，而不是要求推翻现有 Worker。

### 6.3 目标 Modal 拆分

| Stage | 资源 | Checkpoint |
|---|---|---|
| ingest/mesh inspect | CPU | normalized mesh + UV report |
| render views | GPU/EGL | multi-view bundle |
| generate texture views | L40S | texture samples |
| bake/backproject | GPU | baked texture + mesh state |
| export/validate | CPU | GLB/OBJ bundle/report |

### 6.4 关键风险

- OBJ 外部依赖/路径；
- 无 UV/退化 UV；
- multi-view 大中间产物；
- nvdiffrast/gsplat release ABI；
- 4K/更大纹理超 Web budget；
- mesh topology 或 transform 被无意改变；
- 中文/英文 prompt 模型差异。

### 6.5 AgentScape 价值

用于把 geometry-only Worker 输出升级为 textured GLB。原 Asset ID 不应被静默替换；结果形成新 revision，并保留 `derived_from`。

## 7. Asset Convert 详细映射

### 7.1 上游行为

`embodied_gen/data/asset_converter.py` 通过 converter 将 EmbodiedGen asset/URDF 输出到不同 simulator：

- `MeshtoMJCFConverter`：MuJoCo/Genesis；
- `MeshtoUSDConverter`：IsaacSim/USD；
- `PhysicsUSDAdder`：给 USD 增加 physics；
- URDF 可直接供 SAPIEN/PyBullet/IsaacGym。

### 7.2 Modal 首版范围

1. 验证和重新打包 URDF + mesh dependencies；
2. 产出 normalized GLB 供 AgentScape；
3. 提取 visual/collision/scale/origin/inertial metadata；
4. 可选 MJCF/USD 作为下载结果；
5. 独立 validation report。

### 7.3 不可混淆的事实

- 格式可解析不等于物理准确；
- USD 没有 physics postprocess 时必须标明；
- URDF collision mesh 若只是 visual fallback，不可声称“accurate collision”；
- AgentScape 使用 Rapier，不直接执行 URDF/MJCF joint，需要 Compiler 转换证据。

## 8. Affordance 详细映射

### 8.1 上游输入

一个 self-contained URDF bundle，至少有 visual mesh、collision mesh、category/physical metadata。

### 8.2 上游阶段

1. `part_seg.py`：functional part segmentation；
2. `partsemantics_annot.py`：LLM part semantics；
3. `gen_grasp.py`：6-DoF grasp proposals；
4. `eval_grasps.py`：SAPIEN physical filtering；
5. `affordance_annot.py`：编排与 URDF custom_data 更新。

阶段依赖严格：semantic 依赖 segmentation；grasp 依赖 semantic；evaluation 依赖 proposal。

### 8.3 输出

- `mesh_part_seg.glb`，包含颜色分区和 face IDs metadata；
- `affordance_annot.json`，包含 part name、graspable、scenario/confidence、functional labels、description、grasp poses；
- 指向这些文件的更新后 URDF。

### 8.4 Evidence 分级

| Evidence | 可支持的结论 | 不可支持的结论 |
|---|---|---|
| segmentation | faces 属于候选 part | part 有真实 joint |
| semantic annotation | LLM 建议的名称/用途 | 动作可执行 |
| grasp proposal | 候选 6-DoF pose | 抓取一定成功 |
| SAPIEN filtered grasp | 在指定仿真设置通过 | AgentScape/Rapier 中已验证 |
| URDF custom data | bundle 有引用 | 浏览器已正确加载 |

AgentScape Adapter 必须保留此分级。

## 9. Background Scene 详细映射

### 9.1 上游行为

`scripts/gen_scene3d.py`：

1. prompt 生成 panorama；
2. panorama → colored mesh；
3. gsplat training；
4. 导出 `pano_image.png`、`mesh_model.ply`、`gs_model.ply`、`gsplat_cfg.yml`、`video.mp4`。

默认 4000 steps，文档给出的典型时间约 30 分钟。

### 9.2 Modal 计划

- panorama 和 mesh 是可复用 checkpoint；
- 3DGS training 独立长 GPU stage；
- 输出保留 mesh 与 GS 两种 representation；
- 为 mesh 加 PLY→GLB conversion/decimation/texture budget；
- 记录 scene scale、origin、up axis、camera/GS config；
- prompt/seed/model/revision 完整 lineage。

### 9.3 AgentScape 首版

只把受预算约束的 background GLB 作为 environment proposal；3DGS PLY 保存但不加载。背景碰撞需要单独简化/验证，不能用视觉高密 mesh 直接做 Rapier collider。

## 10. Layout 详细映射

### 10.1 上游行为

`scripts/gen_layout.py` 使用 `LayoutGenConfig`，整体包含：

- task description → scene graph；
- 每个对象 Text→Image→3D；
- background resolve；
- BFS/约束布局；
- optional robot insertion；
- `layout.json`、asset3d、background、images、scene tree、Iscene video；
- `compose_layout.py` 可重新组合；
- `simulate_sapien.py` 加载。

### 10.2 Modal 拆分重点

- Planner proposal 与 deterministic composition 分开；
- 每个 asset 是 child Job，可并发但有限流；
- 复用已有 asset/background 优先；
- parent Job 收集 child result/admission；
- layout artifact 引用 hash/asset IDs，不只写易失路径；
- robot insertion 作为显式 option；
- partial failure 有明确列表和重试策略。

### 10.3 到 AgentScape 的转换缺口

EmbodiedGen `layout.json` 不能直接当 WorldSpec v1。需要映射/扩展：

- stable instance/asset identity；
- translation、rotation、scale；
- coordinate/unit；
- background/environment；
- robot/entity distinction；
- ON/NEAR 之外的 relation；
- collision/physics provenance；
- world bounds/navigation hints。

最终位置仍由 AgentScape Physics/World Admission 验证。

## 11. Room/House 详细映射

### 11.1 上游行为

`scripts/room_gen/gen_room.py`：

- explicit room type 或 GPT router；
- seed/complexity/House large-scene；
- Infinigen/Blender generation；
- generation、URDF export、USD export 可分别开关；
- 输出 raw Blender、whole-room URDF、per-instance meshes、USD/textures；
- USD 需要 physics postprocess 才能称物理就绪。

### 11.2 Modal 特殊约束

- 必须使用含 `bpy` 的固定 Blender Python；
- 子进程、临时磁盘、文件数量和 image size 需要先做可行性 benchmark；
- GPT routing Secret 与 deterministic explicit config 分开；
- Blender raw scene 是重要 checkpoint；
- export 可以在生成失败之外单独重试；
- House 与 detail 可能需要不同资源/timeout；
- 不与 asset-core 镜像合并。

### 11.3 AgentScape 接入

需要 Environment Bundle Adapter：

- room shell/background visual；
- static collision simplification；
- per-instance asset references/poses；
- bounds、ground、spawn points；
- navigation rebuild hints；
- texture/resource budget；
- dynamic vs baked instances。

因此它晚于单资产与 layout 接入。

## 12. Simulation 与 Robot Learning

上游 `simulate_sapien.py`、`parallel_sim.py`、grasp evaluation、Genesis cloth 等能力很广。Modal 化前必须为每个用例回答：

- 是离线验证、数据生成还是交互式服务；
- 结果由谁消费；
- 是否需要实时 GPU rendering；
- 是否有许可/驱动/容器约束；
- determinism 与 seed；
- 输出 artifact/metric；
- 与 AgentScape Rapier 的证据如何映射。

没有明确消费者时不进入主路线。

## 13. Secret、网络与许可矩阵

| 能力 | 可能的外部依赖 | 计划处理 |
|---|---|---|
| HUNYUAN3D | Tencent Secret/COS 网络 | 独立 backend/Secret，默认不启用 |
| Text→Image | HF gated/license 或固定 Kolors | exact revision、离线权重、许可记录 |
| Semantics/Layout/Room routing | GPT provider Secret | 只注入相关 CPU/control worker，prompt 隐私政策 |
| Weight preload | HF/ModelScope/GitHub | CPU sync，有网络；GPU runtime offline |
| Blender/Infinigen | 大量依赖/许可 | 独立 image、SBOM、可行性 gate |
| SAPIEN/Genesis/Isaac | simulator/runtime license | 独立可选 deployment |

## 14. 上游升级审计

升级 EmbodiedGen commit 时按能力逐项检查：

1. public entrypoint/参数变化；
2. output tree/schema 变化；
3. submodule commits；
4. CUDA extension ABI；
5. model weight/revision变化；
6. headless patch 是否仍需；
7. license/Secret 新要求；
8. fixed fixtures diff；
9. 冷启动/显存/输出 budget；
10. AgentScape adapter fixture compatibility。

升级不是一次全仓替换；每个已部署 workflow 必须重新 canary 后才切 revision。
