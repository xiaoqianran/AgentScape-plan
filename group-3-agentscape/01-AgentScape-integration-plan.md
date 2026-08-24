# `AgentScape` 对接 Modal 与 EmbodiedGen 的实施计划

## 1. 项目定位

AgentScape 是最终消费与验证层。它不把“生成模型返回了一个 GLB”当作世界完成，而是维护下面的事实链：

```text
Provider proposal/artifacts
→ authenticated bytes
→ Asset Compiler
→ Asset Manifest + evidence
→ Asset Admission
→ WorldSpec proposal
→ deterministic composition/instantiate/relations
→ Physics/Navigation/Validation/Repair
→ World Admission
```

Modal、EmbodiedGen、LLM 都是 provider；AgentScape Runtime 才是可执行几何、物理、导航、关系与动作完成的最终事实源。

## 2. CodeGraph 当前事实

### 2.1 已有资产接入骨架

- `src/assets/gateway/HttpAssetGenerator.js`：120 秒同步 JSON POST。
- `src/assets/library/AssetLibrary.js`：reuse-first search/resolve；可接 `{manifest}` 或 recognized provider payload。
- `src/adapters/EmbodiedGenAdapter.js`：松散 EmbodiedGen payload → GLB Manifest。
- `src/compiler/AssetCompiler.js`：可直接接 `bytes`，依次完成 GLTF inspect、normalize、geometry、semantic/articulation proposal、collider、remote enrichment、segmentation、part/joint/collision、optimization、resource budget、quality、manifest。
- `src/assets/admission.js`：明确 `ready/provisional/rejected`。
- `src/assets/storage/CompiledAssetStore.js`：编译 bytes 进入 IndexedDB。

### 2.2 已有世界主链

`createWorldPipeline.js` 当前固定：

```text
normalize_spec
→ resolve_assets
→ asset_admission
→ compose_layout
→ instantiate
→ apply_relations
→ validate
→ repair
→ finalize
```

`runWorldPipeline` 总是执行完整 canonical pipeline；world rejected 时恢复调用前 scene。`WorldComposer.js` 已基于 root collider footprint、环境 bounds/ground 和 Rapier poseClear 做确定性布局；articulated root-only coverage 会降级为 provisional。

### 2.3 当前 contract 限制

`WorldSpec` v1 只有：

- assets：id/assetId/query/prompt/type/position/generate/provider；
- relations：ON/NEAR；
- generation default。

它缺 rotation、scale、environment、coordinate metadata、更多关系和 provider result references，不能直接承载 EmbodiedGen layout/room。

### 2.4 当前 EmbodiedGen Adapter 的真实含义

当前 Adapter：

- 要求浏览器可达 GLB URL；
- dimensions/mass/friction 缺失时使用默认；
- 建立 conservative box collider；
- 只将 provider 的 pickup/place 部分映射为动作；
- 将语义标为 unverified；
- admission 固定 provisional，原因包括 fallback collider 与 unverified semantics。

这个 fail-closed 行为是正确的起点，但不足以消费 GLB+URDF+collision+part evidence 的完整 bundle。

### 2.5 不可回退的 Runtime 事实

AgentScape 已有成熟的：

- Manifest schema/part/joint/target/collider 门禁；
- Rapier physics 与真实 action completion；
- Recast navigation；
- `ready/provisional/rejected`；
- compile/import/generate/spawn/world pipeline skills；
- trace、history、rollback、world validation/repair。

Modal 接入不得绕过这些门，也不能把 provider semantic affordance 直接加入可执行 `open/close`。

### 2.6 已有的有界 World retry

最新工作区新增 `src/pipeline/WorldRetry.js`，并已接入 `runWorldPipeline`/core skill：

- 从 asset/layout/relation/validation admission 报告提取结构化 findings；
- 只有“catalog 缺失、generator 已配置、原请求尚未允许生成”才可自动构造 `enable-generation` action；
- 默认 attempt/budget 有上限；任何不可重试 finding 都停止；
- `ToolCallingAgent` 记录已尝试 WorldSpec identity，阻止同一计划在同一任务中原样重复提交；
- world-provisional/rejected 仍不能被 Agent 声称为完成。

这正好可以作为后续 Modal 资产生成的 retry policy 起点，但目前 action 只有泛化的 `generate=true`，还没有 provider/strategy/cost/interaction/Job identity，也不能处理 Modal 异步失败。计划必须扩展这条既有门，而不是在 Connector 或 UI 旁边再造一个无预算的重试循环。

## 3. 目标架构

```text
AgentScape UI / Agent Skills
          │
          ▼
ProviderRegistry
  ├── catalog/local assets
  ├── modal-2d text-to-image
  ├── modal-3d models
  └── EmbodiedGen workflows
          │
          ▼
LocalConnectorClient ── paired loopback session ── modal-3D-client Python Agent
          │                                      ├── Modal credentials
          │                                      ├── durable jobs
          │                                      └── artifact streaming
          ▼
GenerationJobStore / Events
          │
          ▼
ArtifactImporter（descriptor → bytes → hash）
          │
          ├── Generic GLB → AssetCompiler
          ├── EmbodiedGen Asset Bundle → evidence-aware compile
          └── World/Environment Bundle → adapter → WorldSpec proposal
          ▼
Asset Admission → World Pipeline → World Admission
```

## 4. 核心原则

1. **Browser 不持有 Modal Secret**：只持短期、最小 scope 的本地 Connector token。
2. **Job 是异步资源**：3～30 分钟任务不能由一个 120 秒 fetch 表达。
3. **Artifact 以 bytes/hash 进入 Compiler**：临时 URL 只是 transport。
4. **Provider evidence 不越权**：semantic、collision、grasp、joint 各有 evidence level。
5. **Reuse first**：本地已编译资产、相同 hash、相同 request 优先复用。
6. **Compiler/Admission 不可跳过**：即使 EmbodiedGen 返回“sim-ready”。
7. **World proposal 与 world truth 分开**：layout/room 先转 proposal，再由 Runtime 验证。
8. **失败可回滚**：生成的远端 Job 可保留，但 rejected world 不污染 live scene。
9. **策略显式**：组合式 2D→3D 与 EmbodiedGen Text→3D 是不同 capability/request，不静默互相 fallback。

## 5. 文件级影响范围

| 当前文件/模块 | 计划动作 |
|---|---|
| `src/core/JsonGateway.js` | 保留给普通 JSON 服务；不强行塞入 Job/SSE/bytes/配对逻辑 |
| `src/assets/gateway/HttpAssetGenerator.js` | 兼容旧同步 provider；新增 async provider interface，不再作为 Modal 主入口 |
| `src/runtime/WorldRuntime.js` | 注入 ProviderRegistry、ConnectorClient、GenerationJobStore；不保存 Secret |
| `src/assets/library/AssetLibrary.js` | resolution policy 支持 async job、compiled hash dedup、provider selection |
| `src/adapters/EmbodiedGenAdapter.js` | 升级为 bundle/evidence adapter；保留 loose legacy adapter 路径 |
| `src/compiler/AssetCompiler.js` | 使用已有 bytes 入口；接收 provider evidence/artifact bundle，不负责远端认证 |
| `src/assets/schema.js` | 增加 provenance/evidence/derived lineage 的验证；不降低 part/joint门禁 |
| `src/assets/admission.js` | 组合 provider evidence、compiler quality、runtime verification，明确 reason codes |
| `src/assets/storage/CompiledAssetStore.js` | 保存 source hash、bundle lineage、provider revision、artifact metadata |
| `src/pipeline/WorldSpec.js` | 分阶段演进到 v2；兼容 v1 normalize |
| `src/pipeline/createWorldPipeline.js` | async resolution/fan-out、environment proposal、provider result refs、admission |
| `src/pipeline/WorldRetry.js` | 保留现有 bounded/missing-only 基线；扩展 provider strategy、linked request、预算与异步失败 finding，不允许原样重提 |
| `src/pipeline/WorldComposer.js` | 支持 rotation/scale/environment bounds；继续 deterministic + physics check |
| `src/skills/registerCoreSkills.js` | 增加 provider/job/import bundle skills；长任务不伪装成同步 mutation |
| `src/persistence/SceneSerializer.js` | 保存 asset hash/provider/workflow lineage 与 environment bundle refs |
| `src/observability/TraceRecorder.js` | 加 connector/job/artifact/compiler/world correlation |
| UI/main/settings | 配对、provider 状态、Job Center、资产审阅、世界生成进度 |

建议新增独立目录：

```text
src/providers/
  ProviderRegistry.js
  LocalConnectorClient.js
  Modal3DProvider.js
  EmbodiedGenProvider.js
  GenerationJobStore.js
  ArtifactImporter.js
src/adapters/
  EmbodiedGenAssetBundleAdapter.js
  EmbodiedGenLayoutAdapter.js
  EmbodiedGenEnvironmentAdapter.js
```

## 6. 工作包

### AS-00：当前 Runtime/世界生成基线

任务：

1. 冻结现有 `WorldSpec→World Admission` tests。
2. 固定 raw EmbodiedGen Adapter provisional fixture。
3. 固定 `AssetCompiler.compile({bytes})` fixture。
4. 记录当前 environment bounds、root collider composer、rollback 行为。
5. 对工作区已有未提交改动只读审计，后续实施避免覆盖。

验收：增加 provider 前，现有静态世界、Agent skills、physics/navigation 测试全部可比较。

### AS-01：Provider Registry

建立 provider 与 capability 抽象：

- provider identity/version/health；
- operation/workflow capabilities；
- input/output schema；
- auth mode；
- async job support；
- cost/duration class；
- artifact transport；
- status/deprecation。

首批 provider：

1. `local-catalog`；
2. `legacy-http-generator`；
3. `modal-2d`；
4. `modal-3d`；
5. `embodiedgen`。

AssetLibrary 不再靠 `options.provider === 'embodiedgen'` 的散落条件判断，而向 Registry 请求满足 operation/input/output 的 provider。

### AS-02：Local Connector 配对客户端

实现产品级客户端行为：

1. 用户输入/发现 Connector 地址只允许 loopback。
2. 显示 connector identity/version；用户在 Tauri 侧确认配对。
3. AgentScape 获得短期 token、scope、expires。
4. token 只存在内存或明确受控 session storage；页面重启需重新授权或使用安全的 Tauri 桥，不写 localStorage。
5. health/capability version negotiation。
6. revoke/expiry/connector restart 有明确 UI。
7. 防止静默连接任意局域网地址或把 artifact token发送到第三方。

AgentScape 只请求 `capabilities/jobs/artifacts` scope，不获得 credential management。

### AS-03：Async Generation Job Store

Browser 侧保存的是 Connector Job 的投影，不成为远端主事实。状态：

- submitting/queued/running/cancel_requested；
- connection_required；
- downloading/compiling；
- succeeded/provisional/rejected/failed/cancelled/expired。

功能：

- submit/get/list/cancel；
- SSE event sequence + reconnect，轮询 fallback；
- 页面刷新后从 Connector list 恢复；
- parent/child Job；
- progress 只展示服务端声明 stage，不编造百分比；
- 终态触发 artifact import，但可由用户选择“只生成、不导入”；
- Job Center 与当前 world mutation 解耦。

### AS-04：Artifact Importer

职责：

1. 读取 result/artifact manifest；
2. 按 role 选择 required artifacts；
3. 流式 fetch，验证 content-length/hash/MIME；
4. 限制 bytes，拒绝 zip bomb/路径逃逸；
5. GLB 直接形成 `Uint8Array` 交 Compiler；
6. JSON/URDF/OBJ bundle 解析前保持不可信；
7. 相同 SHA-256 命中已有 compiled asset；
8. 临时 bytes/object URL 用后释放；
9. trace 记录 artifact ID/hash，不记录 signed token。

如果 Connector 支持 range，Importer 可恢复大文件下载；hash 不匹配时不进入 Compiler。

### AS-05：Generic `modal-3D` 单资产链

首个 E2E operation：`modal-3d.asset.image_to_3d.v1`。

流程：

1. 选择本地图片或复用 modal-3D-client 的 canonical artifact；
2. 通过 Connector 提交 model/profile/seed；
3. 等待 Job；
4. 下载 primary GLB bytes；
5. `AssetCompiler.compile({bytes, sourceName, assetId, label})`；
6. compiler quality → Asset Admission；
7. 注册 manifest；
8. 用户/Agent 可 spawn 或进入 WorldSpec。

Generic 3D Worker 不提供 URDF/affordance 时，Compiler fallback collider 可能导致 provisional；这是正确结果，不应为了“成功率”绕开门禁。

### AS-05A：组合式 `modal-2d → SAM → modal-3D`

AgentScape 通过统一客户端 Connector 创建 parent pipeline：

1. Prompt 经过可选本地优化并确认；
2. `modal-2d` child Job 生成 lossless primary；
3. interactive/auto policy 决定接受、重生成或从 batch 选图；
4. Cloud SAM/skip 形成 canonical RGBA；
5. `modal-3D` child Job 生成 GLB；
6. authenticated bytes → Compiler → Admission。

3D stage 失败时复用 2D/SAM artifact，不重复前置费用。该路线与 EmbodiedGen Text→3D 的选择、状态、成本和 lineage 详见 `03-dual-generation-strategy-plan.md`。

### AS-06：EmbodiedGen Asset Bundle Adapter

输入不再是松散 `glb_url`，而是：GLB、URDF、collision、validation、lineage、可选 part/affordance evidence。

Adapter 计划：

1. bundle schema/required roles 验证；
2. 解析坐标、unit、source transform；
3. 选 visual GLB bytes；
4. 从 URDF 提取 visual/collision/origin/inertial 的 evidence；
5. collision mesh 只有转换为 AgentScape 支持的 convexHull/primitive 并过 budget 才进入 Manifest；否则交 Compiler fallback；
6. part segmentation 转成 Compiler `partSegmentation` evidence；
7. semantic part records 转成 `partProposal`/provenance；
8. grasp poses 保存在 provider evidence，不变成 AgentScape `pickup` 成功证明；
9. joint 不存在时不能从 `open` semantic 猜 revolute/prismatic；
10. 将 validation/provisional warnings 进入 admission reason。

旧 `EmbodiedGenAdapter.toManifest(payload,{glbUrl})` 保留兼容，但始终 provisional。

### AS-07：Compiler Evidence Bridge

确保上游证据进入正确 pass，而不是直接构造最终 Manifest：

- GLB bytes → GLTFInspect/Structure/Normalize/Geometry；
- part face IDs → SegmentationEvidence/SegmentMaterialize；
- semantic labels → PartProposal/Semantic evidence；
- collision mesh → PartCollider/ArticulatedCollision 的候选输入；
- URDF joint frame（未来）→ JointFrame candidate，仍需 schema/runtime verification；
- provider resource summary → ResourceBudget 的参考，不替代重新检查。

Compiler report 保存 provider evidence accepted/rejected/ignored 的原因，便于调试跨系统差异。

### AS-08：Asset Admission 收敛

建议 admission reason 分层：

```text
provider: input/output validation、collision/semantic evidence
compiler: GLB、geometry、resource、part/joint/collider quality
runtime: articulation trajectory/physics verification
```

组合规则：任何层 rejected 即 rejected；没有 hard 但存在未知/fallback 即 provisional；只有所需层均验证才 ready。

典型 reason：

- `PROVIDER_COLLISION_FALLBACK`；
- `PROVIDER_SCALE_UNKNOWN`；
- `PART_SEMANTICS_UNVERIFIED`；
- `GRASP_ONLY_SAPIEN_VERIFIED`；
- `COMPILER_FALLBACK_COLLIDER`；
- `RESOURCE_BUDGET_ADVISORY`；
- `ARTICULATION_UNVERIFIED`。

### AS-09：Agent Skills 与权限语义

新增只读/长任务工具：

- `listGenerationProviders`；
- `listGenerationCapabilities`；
- `submitGenerationJob`；
- `getGenerationJob`；
- `cancelGenerationJob`；
- `importGenerationResult`；
- `generateAndCompileAsset`（高层 orchestration，可返回 pending）；
- `importEmbodiedGenBundle`；
- `generateWorldProposal`（后置）。

规则：

- submit 是外部副作用/可能计费，需要 policy scope；
- 长任务工具不能保持一个 Tool call 30 分钟；返回 pending Job identity；
- `job-succeeded` 不等于 `asset-ready`；
- import/compile 后返回 `asset-ready/provisional/rejected`；
- `runWorldPipeline` 仍是唯一世界准入工具；
- cancel 不是 world mutation，也不回滚已经存在的 scene。

### AS-10：UI 与 Job Center

产品面至少包含：

- Connector 状态/配对/撤销；
- provider/model/workflow capability browser；
- 输入表单（schema/profile 驱动）；
- cost/duration/Secret prerequisite 提示；
- Job list、stage、parent/children、取消、恢复；
- result artifact/validation/lineage；
- Compile report 与 admission reasons；
- “注册资产”“加入当前世界”“用于 WorldSpec”分开；
- provider success 但 compiler rejected 的可解释页面。

### AS-11：Prompt→WorldSpec Planner

当前 roadmap 的 P0 仍应成立。Planner 只输出 proposal，计划流程：

1. 用户 intent/task；
2. Planner 生成严格 WorldSpec；
3. normalize/schema check；
4. AssetLibrary reuse-first；
5. 只为 missing asset 创建生成 Job；
6. 所有 required asset admission；
7. deterministic composer；
8. runtime validation/repair/finalize。

Planner 不选择任意 Modal function 名；它只能选择 Registry 中 policy 允许的 provider/capability。Prompt 与生成成本 policy 在执行前确定。

### AS-12：WorldSpec v2 演进

在保持 v1 normalize 的基础上增加：

- `schemaVersion`；
- environment/background proposal；
- asset source/result reference；
- rotation quaternion 或明确 Euler convention；
- scale；
- coordinate/unit metadata；
- static/dynamic intent；
- optional constraints/bounds；
- relation 扩展，例如 INSIDE/ATTACHED（只有 Runtime 可验证的先加入）；
- provider lineage/seed；
- required vs optional assets。

每个新字段必须在 normalize、schema、serializer、composer、validator、tests 同时落地，不能只为适配某个 layout.json 临时透传。

### AS-13：EmbodiedGen Layout Adapter

转换步骤：

1. 验证 layout bundle 和所有引用 hash；
2. 建立 provider asset identity → AgentScape catalog/compiled asset；
3. 转单位、轴、transform；
4. 映射 instance ID、pose、type；
5. 映射可验证 relation；不支持关系保留 warning/provenance；
6. background 转 environment proposal；
7. robot/entity 按 AgentScape 支持决定复用/忽略/placeholder；
8. 产出 WorldSpec v2 proposal；
9. 走完整 world pipeline。

provider layout pose 作为 explicit proposal，但 `WorldComposer/Physics` 仍检查 bounds/overlap；失败时可以 bounded repair/recompose，不能直接实例化穿模场景。

### AS-14：Background/Environment Bundle

首版背景使用 mesh→budgeted GLB：

- visual GLB；
- simplified static collision；
- bounds/ground/up axis；
- spawn/camera/navigation hints；
- optional pano/preview；
- 3DGS artifact reference（unsupported）。

Runtime 需要可替换 environment 生命周期：卸载旧 environment colliders/navmesh/render resources，装载新环境，重建 navigation，并允许失败回滚。动态 environment 切换的历史/serializer 行为必须先定义。

### AS-15：Room/House Adapter

Room bundle 处理：

- shell/static environment；
- per-instance visual/collision/pose；
- baked vs editable instance policy；
- resource budget/LOD；
- world bounds/ground/navigation；
- oversized scene rejection/decimation request；
- USD 仅作 provenance/download，不让浏览器直接解析；
- AgentScape 世界保存 provider bundle hash 和 per-instance compiled keys。

这一步依赖 Environment Bundle、WorldSpec v2、layout adapter、navigation rebuild 全部稳定。

### AS-16：持久化与离线恢复

SceneSerializer/LocalSceneStore 保存：

- compiled storage key + GLB hash；
- provider/workflow/model/revision/seed；
- source bundle/artifact IDs/hash；
- admission/verification status；
- environment bundle lineage；
- WorldSpec 与 final placement diff。

打开旧 scene 时：优先 IndexedDB compiled bytes；不因 Connector 不在线而失效。只有缺失 bytes 才提示重新配对/下载，不能自动重新计费生成。

### AS-17：安全边界

- 只连接 `127.0.0.1`/`localhost` 且校验 Connector identity；
- token 不进 localStorage、scene、trace、error；
- artifact URL 不接受任意 host redirect；
- JSON/URDF/OBJ/GLB 都按不可信输入处理；
- bytes、纹理、vertices、draw calls、JSON depth/array 有限制；
- HTML/UI 渲染 prompt/label/error 时转义；
- provider 不能返回任意 Agent skill 名并自动执行；
- submit/cancel/import 权限分别审计；
- 生成结果不含 Secret/绝对云端路径。

### AS-18：观测与故障归因

Trace correlation：

```text
agent/tool call
→ world request
→ connector job
→ provider remote job/stages
→ artifact IDs/hashes
→ compiler passes/quality
→ asset admission
→ world pipeline/admission
```

UI/trace 能区分：Connector unreachable、auth、provider job、artifact transport、hash、compiler、asset admission、composer、physics、navigation、world validation。不能统一显示“生成失败”。

### AS-19：测试计划

**Provider/Connector 单元**

- pairing/version/scope/expiry/revoke；
- capabilities normalization；
- async Job/SSE reconnect/sequence；
- cancel race/connection_required；
- error normalization。

**Artifact Importer**

- stream/hash/length/MIME；
- limit、redirect、truncated、corrupt GLB；
- bundle traversal/dependency；
- same-hash dedup；
- token/object URL cleanup。

**Adapter/Compiler**

- generic modal-3D GLB；
- EmbodiedGen visual+URDF+collision；
- part segmentation/affordance；
- unknown scale/default collision；
- malicious/malformed bundle；
- evidence accepted/ignored reason；
- budget ready/provisional/rejected。

**World**

- v1→v2 normalization；
- async asset resolve；
- multiple generated assets；
- generated asset rejected causes no spawn；
- layout explicit pose blocked；
- world rollback preserves remote Job/history；
- environment swap/nav rebuild/rollback；
- serialized scene offline restore。

**Skills/Agent**

- search before generate；
- submission policy/cost confirmation；
- pending job 不被当 success；
- import后 provisional 正确传递；
- runWorldPipeline canonical order 不可跳过；
- world-rejected rollback。

**真实 E2E**

- modal-3D single asset；
- EmbodiedGen Text→3D asset bundle；
- retexture derived asset；
- affordance evidence compile；
- background environment；
- layout 2～3 assets；
- room minimalist（最终 Gate）。

## 7. 分阶段交付

### A：单资产基础

AS-00～05A：配对、capability、Job、artifact bytes、modal-2d、generic modal-3D、组合式 2D→3D compile/admission。

### B：EmbodiedGen 高价值资产

AS-06～10：asset bundle、evidence bridge、admission、skills、Job Center；接 Image/Text/retexture/convert/affordance。

### C：生成式世界

AS-11～13：Planner、WorldSpec v2、layout adapter、多资产 fan-out。

### D：环境与房间

AS-14～16：background/environment、room、持久化离线恢复。

### E：加固

AS-17～19：安全、观测、故障注入、全链 E2E。

## 8. 完成定义

- AgentScape 不保存 Modal Secret；
- Connector 配对、能力发现、异步 Job、取消/恢复可用；
- 私有 Artifact 以 bytes/hash 进入 Compiler；
- generic image→3D、组合式 modal-2d→modal-3D 与 EmbodiedGen Text→3D 链均通过；
- 两条 Text→3D 策略显式选择，跨策略 fallback 不静默重复计费；
- provider evidence 不绕过 Manifest/Compiler/Admission；
- scene 保存后离线可恢复 compiled assets；
- Prompt→WorldSpec 使用 reuse-first 和受控 provider policy；
- EmbodiedGen layout 能转 proposal 并通过/拒绝于 world pipeline；
- background/room 使用 environment bundle 并重建 navigation；
- `world-ready` 才被 Agent 视为验证完成；
- 所有 rejected world 回滚，不删除有价值的生成 Job/artifact；
- tests 能独立归因 provider、artifact、compiler、physics 与 world failure。

## 9. 非目标

- 在浏览器里实现 Modal credential management；
- 把 EmbodiedGen `layout.json` 直接注入 scene；
- 因 provider 写了 `open` 就自动造 joint/action；
- 首版实现 3DGS renderer；
- 自动无限 fan-out 生成场景中的所有对象；
- Agent 未经 policy/成本确认提交高成本 scene/room workflow。
