# AgentScape 双生成方案：组合式 2D→3D 与 EmbodiedGen Text→3D

> 本文继续作为 **Asset Sourcing Strategy / 资产来源策略** 有效，但不再定义 AgentScape 核心架构推进顺序。未来 Core Gate 以 [`07-agent-native-world-architecture-replan.md`](./07-agent-native-world-architecture-replan.md) 为准；无论走哪条生成方案，都必须汇入同一 Asset Compiler / Admission / World IR / Runtime / Verification 真值链。

## 1. 方案定义

AgentScape 对“从文字生成资产”提供两条并列方案，而不是把不同后端塞进一个含糊的 `generateAsset`：

### 方案 A：组合式 2D→3D

```text
Text intent
→ modal-2d（SANA / Z-Image）
→ lossless primary image
→ 可选用户审阅 / SAM候选 / canonical RGBA
→ modal-3D（4个模型之一）
→ GLB
→ AgentScape Compiler / Admission
```

稳定 operation ID建议：`agentscape.asset.composed_text_to_3d.v1`。

### 方案 B：EmbodiedGen Text→3D

```text
Text intent
→ embodiedgen.asset.text_to_3d.v1
   （当前实现事实：固定Kolors Text→Image → SAM3D pipeline）
→ GLB + URDF + validation + 可选collision/GS/video bundle
→ AgentScape bundle adapter
→ Compiler / Admission
```

AgentScape provider operation直接采用 `embodiedgen.asset.text_to_3d.v1`。

### 方案 C：已有图片直接3D（补充入口）

用户上传/历史2D artifact→SAM/skip→modal-3D。它不是第三种Text→3D策略，而是方案A从中间节点开始。

## 2. 为什么必须显式分开

两条路线即使内部都有“图片→3D”，产品语义也不同：

| 维度 | 方案A：组合式 | 方案B：EmbodiedGen |
|---|---|---|
| 2D模型 | SANA/Z-Image显式可选 | 当前固定Kolors backend |
| 中间图 | 一等artifact，可审阅/替换/复用 | workflow中间artifact，可下载但默认自动继续 |
| 对象选择 | Cloud SAM交互或策略化 | 当前rembg/上游pipeline |
| 3D模型 | modal-3D四模型显式可选 | 当前SAM3D workflow |
| 结果 | 主要GLB/模型metrics | GLB/OBJ/URDF/GS/video/validation bundle |
| 可控性 | 高，可分别重试2D/SAM/3D | 统一工作流，阶段恢复由modal-build管理 |
| sim evidence | 主要依赖AgentScape Compiler fallback | 多URDF/collision evidence，但仍需Compiler |
| 适合 | 视觉探索、模型对比、人工选图 | sim-ready bundle、自动化、后续affordance/convert |

如果统一成一个隐式后端，Agent无法解释成本、阶段、产物、失败和readiness，也无法避免静默fallback重复计费。

## 3. Capability 表达

Provider Registry为两条路线分别发布：

### 方案A capability

- supported 2D models/profiles；
- preprocess modes；
- supported 3D models/profiles；
- interaction requirement：是否需要candidate确认；
- stage列表；
- lossless intermediate retention；
- estimated duration/cost class由子capability组合；
- cancel/retry/reuse支持；
- output=`visual-glb`；
- expected AgentScape admission通常可能provisional。

### 方案B capability

- EmbodiedGen workflow/revision；
- current backend事实（Kolors+SAM3D）；
- input prompt/seed/profile；
- stage列表；
- output bundle roles；
- URDF/collision/validation evidence；
- retention/cost/duration；
- future backend不静默替换。

UI与Agent Tool必须展示当前实现backend，不能只写“AI Text→3D”。

## 4. 统一 Parent Request

AgentScape创建一个本地 generation request：

- request ID/user intent；
- explicit strategy A/B；
- prompt record ID；
- provider/profile/options/seeds；
- required deliverables/readiness；
- budget/interaction policy；
- parent world request（如有）；
- idempotency key。

不同strategy必须产生不同request hash；相同prompt不等于可以共享最终结果。

## 5. 方案A阶段与状态

### A1：Prompt确认

- 可使用本地Prompt Pipeline；
- 记录source/transforms/final confirmed；
- 用户或Agent policy确认才提交2D。

### A2：2D生成

- child Job：provider=`modal-2d`；
- output lossless primary + preview；
- 失败只重试2D；
- 相同成功artifact可复用。

### A3：Image review policy

三种模式：

- `interactive`：用户查看图，确认或重新生成；
- `auto-accept`：只在明确允许且image validation通过时继续；
- `select-from-batch`：先生成N个候选，由用户/独立quality policy选择一个。

首版AgentScape默认interactive或单图auto明确确认；不让LLM凭空声称视觉质量合格。

### A4：Preprocess

- scene segmentation；
- candidate选择；
- materialize canonical RGBA；
- 输入已是合格RGBA时skip；
- selection是可恢复暂停状态。

### A5：3D生成

- child Job：provider=`modal-3d`；
- 输入canonical/primary descriptor；
- 选择model/profile/seed；
- 失败可换模型或同模型重试，不重跑2D/SAM。

### A6：Import/Compile/Admission

- GLB bytes/hash；
- AssetCompiler；
- generic provider evidence；
- ready/provisional/rejected；
- rejected不删除2D/canonical/GLB artifact。

## 6. 方案B阶段与状态

### B1：Prompt确认

可使用同一本地Prompt Pipeline，但target adapter必须是EmbodiedGen workflow，而非SANA/Z-Image adapter。

### B2：EmbodiedGen workflow

- 一个Connector parent remote Job；
- modal-build内部有text2image/rembg/sam3d/mesh/texture/finalize阶段；
- AgentScape观察stage但不自行调度内部Worker；
- generated image是bundle中间artifact，可用于失败恢复/fork。

### B3：Bundle import

- 下载artifact manifest；
- required GLB/URDF/validation；
- optional collision/OBJ/GS/video；
- hash/dependency/coordinate/unit验证。

### B4：Evidence-aware Compiler

- GLB bytes；
- URDF/collision evidence；
- 后续affordance/part evidence；
- Compiler/Admission；
- provider ready不自动提升AgentScape ready。

## 7. 路由政策

### 首版：用户/调用者显式选择

Agent Tool参数要求 `strategy`，UI用可比较卡片展示。没有strategy时：

- 交互UI要求选择；
- Agent按任务要求提出建议并等待允许产生费用；
- 不默认同时提交两条路线；
- 不在失败时静默切另一条。

### 建议规则（不是自动事实）

| 需求 | 推荐 |
|---|---|
| 想先看/挑2D概念图 | 方案A |
| 指定SANA/Z-Image或指定modal-3D模型 | 方案A |
| 已有图片 | 方案A从A3/A4开始 |
| 需要URDF/OBJ/validation bundle | 方案B |
| 后续要affordance/convert/simulator | 方案B |
| 只要视觉prop且接受Compiler fallback | 方案A |
| 完全自动批量sim-ready候选 | 先评测后倾向方案B，仍需admission |

### 后续：证据驱动Router

只有在统一benchmark有足够样本后，才可按asset category/required output/cost/latency/admission success做推荐。Router输出recommendation和理由，最终request仍记录显式strategy。

## 8. 失败与Fallback

| 失败点 | 默认动作 |
|---|---|
| Prompt AI | 让用户确认原文/重试，不提交GPU |
| 方案A 2D失败 | retry/换2D model |
| 方案A图不满意 | 从A2重跑，保留旧图 |
| SAM无候选 | refine/换concept/使用direct image/重生成2D |
| 方案A 3D失败 | 从A5重试/换3D model |
| 方案A Compiler rejected | 调整3D模型/profile或选择方案B，需显式新request |
| 方案B text2image失败 | workflow stage retry |
| 方案B reconstruction失败 | 从checkpoint retry，不重跑text image |
| 方案B bundle不完整 | artifact/stage修复；不以GLB URL绕过 |
| 方案B Compiler rejected | 保留bundle；显式选择retexture/convert/方案A |

跨方案fallback总是创建linked request `fallback_of`，展示额外预计成本并要求policy允许。

## 9. Artifact/Lineage 图

### 方案A

```text
PromptRecord
  └→ Job(2D, model/revision/seed)
      └→ Image(primary hash)
          └→ SAM selection
              └→ Canonical RGBA(hash)
                  └→ Job(3D, model/revision/seed)
                      └→ GLB(hash)
                          └→ CompiledAsset(key/admission)
```

### 方案B

```text
PromptRecord
  └→ EmbodiedGen Workflow Job(revision/seed)
      ├→ Generated Image(hash)
      ├→ GLB/OBJ/Texture/GS
      ├→ URDF/Collision evidence
      └→ Validation/Lineage
          └→ CompiledAsset(key/admission)
```

Scene serialization只保存compiled key/hash和provider lineage，不保存临时Connector token/URL。

## 10. Agent Skills

建议高层工具：

- `listAssetGenerationStrategies`；
- `estimateAssetGeneration`；
- `submitComposedTextTo3D`；
- `submitEmbodiedGenTextTo3D`；
- `getGenerationPipeline`；
- `selectGeneratedImage`；
- `selectSegmentationCandidate`；
- `retryGenerationStage`；
- `importGenerationResult`。

工具结果状态：

- `generation-pending`；
- `awaiting-user-selection`；
- `provider-succeeded`；
- `asset-ready/provisional/rejected`。

只有最后三种asset admission中的`asset-ready`才可作为verified asset；provider-succeeded仍需compile。

## 11. World Pipeline集成

Prompt→WorldSpec时每个missing asset可指定strategy，但执行政策：

1. reuse existing compiled asset；
2. dedup相同active request；
3. 根据required output给出推荐；
4. 用户/Policy批准成本；
5. 创建child generation pipeline；
6. 等待所有required asset admission；
7. 才进入compose/instantiate；
8. optional asset失败可按WorldSpec policy省略；
9. world rejected不删除generation artifacts。

同一WorldSpec默认不同时对每个对象跑两种strategy做竞赛；对比生成是独立明确模式。

## 12. 对比评测计划

建立固定prompt/category集：

- common prop；
- furniture；
- thin structure；
- symmetric object；
- transparent/reflective intent；
- articulated-looking object；
- stylized/realistic；
- 中文/英文；
- AgentScape task-relevant support/container objects。

指标：

| 层 | 指标 |
|---|---|
| Prompt/2D | prompt adherence、foreground separability、selection success |
| Provider 3D | job success、time/cost、GLB parse、geometry/texture、reproducibility |
| Bundle | URDF/collision/refs/coordinate completeness |
| Compiler | ready/provisional/rejected、reason distribution、resource budget |
| Runtime | spawn/physics stable、navigation obstacle、pickup/place/open evidence |
| World | placement success、world admission、task completion |

不能只用截图主观比较。每个结果保存strategy、versions、seeds、artifact hashes和人工评分。

## 13. 成本与并发

- 方案A成本是2D + SAM + 3D + transport/compile；
- 方案B成本是EmbodiedGen内部阶段总和；
- estimate显示范围/class，不用旧Kaggle/Modal单样本当SLA；
- batch/world generation有全局并发/预算；
- same image/prompt checkpoint reuse降低重复成本；
- `compare-both`明确会产生两条计费链；
- cancel parent只取消未终结remote jobs，已完成artifacts保留。

## 14. UI计划

生成策略选择卡：

- 名称与真实backend；
- 推荐用途；
- stages；
- 输出类型；
- estimated duration/cost class；
- 是否需要中间选择；
- expected admission evidence；
- current capability health。

Pipeline视图展示children和中间artifacts。方案A用户可在image/candidate阶段暂停；方案B可查看内部stage但不在AgentScape重复控制modal-build DAG。

## 15. 验收

### DS-01 方案A完整链

Prompt→SANA lossless PNG→SAM candidate→Pixal/FastSAM等选定3D→GLB→Compiler→admission；重启每stage可恢复。

### DS-02 方案A重试边界

3D失败后换模型，不重跑2D/SAM；lineage保留多个3D attempts。

### DS-03 方案B完整链

Prompt→EmbodiedGen Text Job→bundle→evidence adapter→Compiler→admission。

### DS-04 显式策略

相同prompt选择A或B得到不同request/lineage；未选择不提交；失败不静默提交另一条。

### DS-05 World missing asset

WorldSpec reuse失败→按指定strategy生成→asset admission→compose；rejected asset阻止spawn并回滚world。

### DS-06 对比模式

用户明确批准后两条并行/顺序执行；结果以独立资产候选展示，不自动挑“成功HTTP”的一个。

### DS-07 离线恢复

Connector离线后已compiled两类资产仍可打开/spawn；缺远端artifact不自动重跑。

## 16. 完成定义

- 两条strategy在capability、UI、Tool和lineage中明确分开；
- 方案A贯通统一客户端2D→SAM→3D；
- 方案B贯通EmbodiedGen Text workflow/bundle；
- 两者都经同一Compiler/Admission；
- failure/retry/checkpoint不重复无关阶段；
- 跨方案fallback显式、可审计、需预算权限；
- Agent与用户能看到真实backend和输出差异；
- 有固定评测集支持未来推荐Router；
- world pipeline只消费admitted asset，不消费provider success口号。

## 17. 非目标

- 首版自动训练质量Router；
- 每次请求默认跑两条路线；
- 用EmbodiedGen `ready`跳过AgentScape验证；
- 用2D preview有损图作为方案A输入；
- 将方案A的多stage实现藏成不可恢复同步fetch；
- 把两条路线输出强行规范成丢失证据的最小GLB URL。
